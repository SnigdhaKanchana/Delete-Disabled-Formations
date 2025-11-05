import groovy.transform.Field
import java.util.regex.Pattern

@Field def matchingVersionFormations = []
@Field def versionMismatchFormations = []
@Field def noMatchingFormations = []
@Field def clusterList = []
@Field def hardDeleteFormations = []
@Field def DISABLED_AGE_THRESHOLD_DAYS = 30
@Field def DISABLED_AGE_THRESHOLD_MINUTES = DISABLED_AGE_THRESHOLD_DAYS * 1440

def timestamp() { new Date().format("yyyy-MM-dd HH:mm:ss") }

def log(msg, level = "INFO") { echo "[${level}][${timestamp()}] ${msg}" }

def validateParam(name, value) {
    log("Validating parameter '${name}'")
    if (!value?.trim()) {
        log("Parameter '${name}' is missing or empty", "ERROR")
        error "[ERROR] ${name} should not be empty"
    }
    log("'${name}' is set to: ${value}")
}

node('infra-base') {
    stage("Validate parameters") {
        log("Starting parameter validation")
        validateParam("TARGET_ENV", params.TARGET_ENV)
        validateParam("CLUSTER_MODE", params.CLUSTER_MODE)
        validateParam("FORMATION_MODE", params.FORMATION_MODE)
        validateParam("DATABASE_TYPE", params.DATABASE_TYPE)
        validateParam("DATABASE_VERSION", params.DATABASE_VERSION)
        if (params.CLUSTER_MODE == "Specific") validateParam("CLUSTER", params.CLUSTER)
        if (params.FORMATION_MODE == "Specific") validateParam("FORMATION", params.FORMATION)

        if (params.FORMATION_MODE == "All" && params.FORMATION?.trim()) {
            error("FORMATION ID must not be entered when FORMATION_MODE is 'All'")
        }
        if (params.FORMATION_MODE == "Specific" && !params.FORMATION?.trim()) {
            error("FORMATION ID must be provided when FORMATION_MODE is 'Specific'")
        }

        def dbType = params.DATABASE_TYPE.trim().toLowerCase()
        def dbVersion = params.DATABASE_VERSION.trim()
        def versionPatternPostgres = ~/^\d+\.\d+$/
        def versionPatternOthers = ~/^\d+\.\d+\.\d+$/

        if (dbType == "postgresql") {
            if (!(dbVersion ==~ versionPatternPostgres)) {
                error "[ERROR] Invalid DATABASE_VERSION format for PostgreSQL"
            }
        } else {
            if (!(dbVersion ==~ versionPatternOthers)) {
                error "[ERROR] Invalid DATABASE_VERSION format for ${dbType}"
            }
        }
        log("DATABASE_VERSION format is valid: ${dbVersion}")

        if (!params.DELETE_CONFIRMATION?.toBoolean()) {
            log("DELETE_CONFIRMATION not checked — job will run in dry run mode only")
        }
    }

    stage("Setup CLI") {
        log("Installing and setting up CLI tool")
        cliTool.install()
        cliTool.setup()

        def clusterMap = new groovy.json.JsonSlurper().parseText(CLUSTER_MAP_JSON)
        clusterList = params.CLUSTER_MODE == "All"
            ? (params.TARGET_ENV == "dev" ? clusterMap.values()[0] : clusterMap[params.TARGET_ENV] ?: [])
            : params.CLUSTER.tokenize(',').collect { it.trim() }

        echo "[INFO] Final clusters list: ${clusterList}"
    }

    def selectedDbType = params.DATABASE_TYPE.trim().toLowerCase()
    def selectedDbVersion = params.DATABASE_VERSION.trim()
    def requestedFormations = params.FORMATION_MODE == "Specific"
        ? params.FORMATION.tokenize(',').collect { it.trim() }
        : []

    def baseUrl = "https://resource-controller.example.com"

    for (cluster in clusterList) {
        stage("Cluster: ${cluster}") {
            log("Processing cluster: ${cluster}")
            cliTool.discover(cluster)
            cliTool.run("cl ${cluster}")

            def fsRaw = sh(script: "cliTool fs | grep -v NAME | awk '{print \$2, \$3, \$4}'", returnStdout: true).trim()
            def formationMetadataList = fsRaw.readLines().collect { line ->
                def parts = line.split(/\s+/)
                parts.size() == 3 ? [name: parts[0], dbType: parts[1].toLowerCase(), version: parts[2]] : null
            }.findAll { it != null }

            def filteredFormations = formationMetadataList.findAll { f ->
                f.dbType == selectedDbType && f.version == selectedDbVersion
            }

            def formationsToProcess = (params.FORMATION_MODE == "All"
                ? filteredFormations*.name
                : filteredFormations.findAll { requestedFormations.contains(it.name) }*.name
            )

            def mismatchedFormations = formationMetadataList.findAll { f ->
                f.dbType == selectedDbType && f.version != selectedDbVersion &&
                (params.FORMATION_MODE == "All" || requestedFormations.contains(f.name))
            }

            mismatchedFormations.each { f ->
                versionMismatchFormations << [
                    cluster: cluster, formationId: f.name, dbType: f.dbType,
                    actualVersion: f.version, expectedVersion: selectedDbVersion
                ]
            }

            if (filteredFormations.isEmpty()) noMatchingFormations << cluster

            for (formationId in formationsToProcess) {
                log("Processing formation: ${formationId}")
                def formationOutput = cliTool.run("formation ${formationId}").replaceAll(/\x1B\[[0-9;]*[mK]/, "")
                def disablementsLine = formationOutput.readLines().find { it.trim().startsWith("Disablements:") } ?: ""
                def disablementTags = disablementsLine.replace("Disablements:", "").trim()
                    .tokenize(',').collect { it.trim() }.findAll { it }

                def knownDisablementTags = ["suspended", "eol", "dead-version", "key-inactive", "unwrap-error"]
                def hasKnownDisablementTag = disablementTags.any { knownDisablementTags.contains(it) }

                def ageMatch = (formationOutput =~ /Disabled age:\s+(\d+)([dhm])/).with {
                    it.find() ? [it.group(1).toInteger(), it.group(2)] : null
                }

                def ageInMinutes = null
                def ageReadable = "None"
                if (ageMatch) {
                    def (value, unit) = ageMatch
                    ageInMinutes = unit == 'm' ? value : unit == 'h' ? value * 60 : value * 1440
                    ageReadable = unit == 'm' ? "${value} minute(s)" :
                                  unit == 'h' ? "${value} hour(s)" : "${value} day(s)"
                }

                def status = ""
                def reason = ""

                try {
                    if (hasKnownDisablementTag && ageInMinutes && ageInMinutes > DISABLED_AGE_THRESHOLD_MINUTES) {
                        def apiKey = credentials("API_KEY")
                        def token = sh(script: "get-token ${apiKey}", returnStdout: true).trim()
                        def apiUrl = "${baseUrl}/v2/resource_instances/${formationId}?recursive=true"

                        if (!params.DELETE_CONFIRMATION?.toBoolean()) {
                            status = "Dry run — would delete"
                            reason = "Disabled for ${ageReadable}, tags: ${disablementTags.join(', ')}"
                        } else {
                            def deleteCommand = """curl -X DELETE '${apiUrl}' \
                                -H "Authorization: Bearer ${token}" \
                                --write-out '%{http_code}' \
                                --silent --output delete_response.json"""

                            def httpCode = sh(script: deleteCommand, returnStdout: true).trim()
                            def responseBody = fileExists('delete_response.json') ? readFile('delete_response.json') : "No response"

                            switch (httpCode.toInteger()) {
                                case 200..299:
                                    status = "Deleted"
                                    reason = "Successfully deleted"
                                    break
                                case 404:
                                case 410:
                                    sh(script: "cliTool delete-formation ${formationId}")
                                    status = "Deleted via CLI"
                                    reason = "Fallback deletion"
                                    hardDeleteFormations << [cluster: cluster, formationId: formationId, httpCode: httpCode]
                                    break
                                default:
                                    status = "Flagged"
                                    reason = "Deletion failed: ${responseBody}"
                            }
                        }
                    } else {
                        status = "Skipped"
                        reason = "Does not meet deletion criteria"
                    }
                } catch (Exception e) {
                    status = "Flagged"
                    reason = "Error: ${e.message}"
                }

                matchingVersionFormations << [
                    cluster: cluster, formationId: formationId, dbType: selectedDbType,
                    version: selectedDbVersion, tags: disablementTags.join(', ') ?: "None",
                    age: ageReadable, reason: reason
                ]
            }
        }
    }

    stage("Summary") {
        echo "\n=== Version Mismatches ==="
        versionMismatchFormations.each { row ->
            echo "| ${row.cluster} | ${row.formationId} | ${row.dbType} | ${row.actualVersion} | ${row.expectedVersion} |"
        }

        echo "\n=== Clusters with No Matches ==="
        noMatchingFormations.each { echo "| ${it} |" }

        echo "\n=== Processed Formations ==="
        matchingVersionFormations.each { row ->
            echo "| ${row.cluster} | ${row.formationId} | ${row.dbType} | ${row.version} | ${row.tags} | ${row.age} | ${row.reason} |"
        }

        if (hardDeleteFormations) {
            echo "\n=== Hard Deletes ==="
            hardDeleteFormations.each { row ->
                echo "| ${row.cluster} | ${row.formationId} | ${row.httpCode} | CLI delete used |"
            }
        }
    }
}
