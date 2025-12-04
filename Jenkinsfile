pipeline {
    agent any

    environment {
        RUN_ID = UUID.randomUUID().toString()
    }

    stages {

        stage('Initialize') {
            steps {
                script {
                    // Global variables for pipeline
                    def SERVICES = []
                    def JAR_PATHS = [:]

                    echo """
                    ===================== PIPELINE START =====================
                    Run ID      : ${RUN_ID}
                    ==========================================================
                    """
                }
            }
        }

        stage('Load Config') {
            steps {
                script {
                    def config = readYaml file: 'services.yaml'
                    SERVICES = config.services   // store globally

                    echo """
                    Loaded Services : ${SERVICES.size()}
                    Services: ${SERVICES*.name}
                    """
                }
            }
        }

        stage('Build Services') {
            steps {
                script {
                    def config = readYaml file: 'services.yaml'
                    SERVICES = config.services

                    // global map to store JAR locations
                    JAR_PATHS = [:]

                    def branches = SERVICES.collectEntries { svc ->
                        ["BUILD-${svc.name}": {
                            node {
                                dir(svc.path) {

                                    echo "▶ Building ${svc.name}"

                                    if (svc.type == "maven") {
                                        withMaven() {
                                            sh "mvn -q clean package -Dpipeline.id=${RUN_ID}"
                                        }
                                    } else if (svc.type == "gradle") {
                                        sh "./gradlew clean build --quiet -PpipelineId=${RUN_ID}"
                                    }

                                    // find jar path
                                    def jar = sh(
                                        script: "find . -name '*.jar' | head -1",
                                        returnStdout: true
                                    ).trim()

                                    // store jar path globally
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
            steps {
                script {
                    def config = readYaml file: 'services.yaml'
                    SERVICES = config.services

                    def branches = SERVICES.collectEntries { svc ->
                        ["TEST-${svc.name}": {
                            node {
                                dir(svc.path) {
                                    echo "▶ Testing ${svc.name}"

                                    if (svc.type == "maven") {
                                        withMaven() {
                                            sh "mvn -q test -Dpipeline.id=${RUN_ID}"
                                        }
                                    } else if (svc.type == "gradle") {
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

        /* ---------------------------------------------------
         * FINAL ARTIFACT SUMMARY (ALL JARS TOGETHER)
         * --------------------------------------------------- */
        stage('Artifacts Summary') {
            steps {
                script {

                    echo """
================== BUILT JAR ARTIFACTS ==================

${JAR_PATHS.collect { svc, jar -> "• ${svc} → ${jar}" }.join("\n")}

==========================================================
"""
                }
            }
        }

    } // stages

    post {
        always {
            echo "======================= PIPELINE END ======================="
        }
    }
}
