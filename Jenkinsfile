def SERVICES = []
def JAR_PATHS = [:]
def CHANGED_SERVICES = []

pipeline {
    agent any

    environment {
        RUN_ID = UUID.randomUUID().toString()
        COMMITS_DIR = "${WORKSPACE}/.last_commits"
    }

    stages {

        stage('Initialize') {
            steps {
                echo """
                ===================== PIPELINE START =====================
                Run ID      : ${RUN_ID}
                ==========================================================
                """
                sh "mkdir -p ${COMMITS_DIR}"
            }
        }

        stage('Load Config') {
            steps {
                script {
                    def config = readYaml file: 'services.yaml'
                    SERVICES = config.services

                    echo """
                    Loaded Services : ${SERVICES.size()}
                    Services: ${SERVICES*.name}
                    """
                }
            }
        }

        stage('Checkout Services') {
            steps {
                script {

                    CHANGED_SERVICES = []
                    sh "mkdir -p ${COMMITS_DIR}"

                    SERVICES.each { svc ->

                        echo "Checking changes for ${svc.name}..."

                        def commitFile = "${COMMITS_DIR}/${svc.name}.txt"
                        def lastCommit = fileExists(commitFile)
                                ? readFile(commitFile).trim()
                                : ""

                        def latestCommit = sh(
                            script: "git ls-remote ${svc.repo} ${svc.branch} | awk '{print \$1}'",
                            returnStdout: true
                        ).trim()

                        if (latestCommit == "") {
                            error "❌ Cannot read latest commit of ${svc.name}"
                        }

                        if (lastCommit != latestCommit) {
                            echo "🔄 CHANGED: ${svc.name}"
                            CHANGED_SERVICES << svc
                        } else {
                            echo "✔ NO CHANGE: ${svc.name}"
                        }

                        writeFile file: commitFile, text: latestCommit
                    }

                    echo """
================ SERVICES WITH CHANGES ================
${CHANGED_SERVICES*.name}
=======================================================
"""
                }
            }
        }

        /* --------------------------
         * BUILD ONLY IF CHANGED
         * -------------------------- */
        stage('Build Services') {
            when {
                expression {
                    def changed = CHANGED_SERVICES.size() > 0
                    if (!changed) {
                        echo "⏭ SKIPPING BUILD: No changed services detected."
                    }
                    return changed
                }
            }
            steps {
                script {

                    JAR_PATHS = [:]

                    def branches = CHANGED_SERVICES.collectEntries { svc ->
                        ["BUILD-${svc.name}": {
                            node {

                                sh "rm -rf build_${svc.name}"
                                sh "git clone -b ${svc.branch} ${svc.repo} build_${svc.name}"

                                dir("build_${svc.name}") {

                                    echo "▶ Building ${svc.name}"
                                    def jar = ""

                                    if (svc.type == "maven") {
                                        withMaven(maven: 'Maven-3.9.11') {
                                            sh "mvn -q clean package -Dpipeline.id=${RUN_ID}"
                                        }

                                        jar = sh(
                                            script: "find target -maxdepth 1 -name '*.jar' | head -1",
                                            returnStdout: true
                                        ).trim()
                                    }

                                    if (svc.type == "gradle") {
                                        sh "chmod +x gradlew"
                                        sh "./gradlew clean build --quiet -PpipelineId=${RUN_ID}"

                                        jar = sh(
                                            script: "find . -path '*/build/libs/*.jar' | head -1",
                                            returnStdout: true
                                        ).trim()
                                    }

                                    JAR_PATHS[svc.name] = jar

                                    echo "✔ Build complete: ${svc.name}"
                                }
                            }
                        }]
                    }

                    parallel branches
                }
            }
        }

        /* --------------------------
         * TEST ONLY IF CHANGED
         * -------------------------- */
        stage('Test Services') {
            when {
                expression {
                    def changed = CHANGED_SERVICES.size() > 0
                    if (!changed) {
                        echo "⏭ SKIPPING TESTS: No changed services detected."
                    }
                    return changed
                }
            }
            steps {
                script {

                    def branches = CHANGED_SERVICES.collectEntries { svc ->
                        ["TEST-${svc.name}": {
                            node {
                                dir("build_${svc.name}") {

                                    echo "▶ Testing ${svc.name}"

                                    if (svc.type == "maven") {
                                        withMaven(maven: 'Maven-3.9.11') {
                                            sh "mvn -q test -Dpipeline.id=${RUN_ID}"
                                        }
                                    }

                                    if (svc.type == "gradle") {
                                        sh "./gradlew test --quiet -PpipelineId=${RUN_ID}"
                                    }

                                    echo "✔ Tests passed: ${svc.name}"
                                }
                            }
                        }]
                    }

                    parallel branches
                }
            }
        }

        /* --------------------------
         * DEPLOY ONLY IF CHANGED
         * -------------------------- */
        stage('Deploy Services') {
            when {
                expression {
                    def changed = CHANGED_SERVICES.size() > 0
                    if (!changed) {
                        echo "⏭ SKIPPING DEPLOY: No changed services detected."
                    }
                    return changed
                }
            }
            steps {
                script {

                    def branches = CHANGED_SERVICES.collectEntries { svc ->
                        ["DEPLOY-${svc.name}": {
                            node {

                                echo "🚀 Deploying ${svc.name}"
                                echo "Using JAR: ${JAR_PATHS[svc.name]}"

                                sh """
                                    echo 'Deploying ${svc.name}...'
                                    echo 'Using artifact: ${JAR_PATHS[svc.name]}'
                                """

                                echo "✔ Deployment complete: ${svc.name}"
                            }
                        }]
                    }

                    parallel branches
                }
            }
        }

        stage('Artifacts Summary') {
            when {
                expression {
                    def changed = CHANGED_SERVICES.size() > 0
                    if (!changed) {
                        echo "⏭ SKIPPING ARTIFACT SUMMARY: No artifacts because no changes detected."
                    }
                    return changed
                }
            }
            steps {
                echo """
================== BUILT JAR ARTIFACTS ==================

${JAR_PATHS.collect { svc, jar -> "• ${svc} → ${jar}" }.join("\n")}

==========================================================
"""
            }
        }

    }

    post {
        always {
            echo "======================= PIPELINE END ======================="

            echo """
==================== FINAL JAR PATH SUMMARY ====================

${JAR_PATHS.collect { svc, jar ->
    "SERVICE : ${svc}\nJAR PATH: ${jar}\n"
}.join("\n")}

================================================================
"""
        }
    }
}
