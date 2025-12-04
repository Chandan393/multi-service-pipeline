def SERVICES = []
def JAR_PATHS = [:]
def CHANGED_SERVICES = []

pipeline {
    agent any

    environment {
        RUN_ID = UUID.randomUUID().toString()
    }

    stages {

        stage('Initialize') {
            steps {
                echo """
                ===================== PIPELINE START =====================
                Run ID      : ${RUN_ID}
                ==========================================================
                """
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

                    SERVICES.each { svc ->
                        echo "Checking changes for ${svc.name}..."

                        def log = sh(
                            script: """
                                git ls-remote ${svc.repo} HEAD | awk '{print \$1}'
                            """,
                            returnStdout: true
                        ).trim()

                        if (!svc.containsKey('last_commit')) {
                            svc.last_commit = "unknown"
                        }

                        if (svc.last_commit != log) {
                            echo "🔄 CHANGED: ${svc.name}"
                            svc.changed = true
                            svc.last_commit = log
                            CHANGED_SERVICES << svc
                        } else {
                            svc.changed = false
                            echo "✔ NO CHANGE: ${svc.name}"
                        }
                    }

                    echo """
================ SERVICES WITH CHANGES ================
${CHANGED_SERVICES*.name}
=======================================================
"""
                }
            }
        }

        /* ----------------------------------------------------
         * UPDATED BUILD BLOCK (Fix: No nested clone folders)
         * ---------------------------------------------------- */
        stage('Build Services') {
            when {
                expression { CHANGED_SERVICES.size() > 0 }
            }
            steps {
                script {

                    JAR_PATHS = [:]

                    def branches = CHANGED_SERVICES.collectEntries { svc ->
                        ["BUILD-${svc.name}": {
                            node {

                                // CLEAN PREVIOUS FOLDER + CLONE OUTSIDE
                                sh "rm -rf build_${svc.name}"
                                sh "git clone -b ${svc.branch} ${svc.repo} build_${svc.name}"

                                // ENTER CORRECT DIRECTORY
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

        stage('Test Services') {
            when {
                expression { CHANGED_SERVICES.size() > 0 }
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

        stage('Deploy Services') {
            when {
                expression { CHANGED_SERVICES.size() > 0 }
            }
            steps {
                script {

                    def branches = CHANGED_SERVICES.collectEntries { svc ->
                        ["DEPLOY-${svc.name}": {
                            node {

                                echo "🚀 Deploying ${svc.name}"
                                echo "Using JAR: ${JAR_PATHS[svc.name]}"

                                // deployment placeholder
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
                expression { CHANGED_SERVICES.size() > 0 }
            }
            steps {
                echo """
================== BUILT JAR ARTIFACTS ==================

${JAR_PATHS.collect { svc, jar -> "• ${svc} → ${jar}" }.join("\n")}

==========================================================
"""
            }
        }

    } // stages

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
