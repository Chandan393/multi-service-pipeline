pipeline {
    agent any

    environment {
        UNIQUE_ID = "${UUID.randomUUID().toString()}"
        JAR_OUTPUT = ""          // Will hold final JAR locations
    }

    stages {

        /* ---------------------------------------------------
         * LOAD YAML CONFIG
         * --------------------------------------------------- */
        stage('Load Config') {
            steps {
                script {
                    def config = readYaml file: 'services.yaml'
                    SERVICES = config.services

                    echo "\n===== PIPELINE INITIALIZED ====="
                    echo "Services Loaded : ${SERVICES.size()}"
                    echo "Run ID          : ${UNIQUE_ID}"
                    echo "================================\n"
                }
            }
        }

        /* ---------------------------------------------------
         * CLEAN BUILD WITH QUIET LOGGING
         * --------------------------------------------------- */
        stage('Build Services') {
            steps {
                script {
                    def tasks = [:]

                    SERVICES.each { svc ->

                        tasks[svc.name] = {
                            node {
                                dir(svc.path) {

                                    echo "\n▶ BUILD START: ${svc.name}\n"

                                    if (svc.type == "maven") {
                                        withMaven(maven: 'Maven-3.9.11') {
                                            sh "mvn -q clean package -Dpipeline.id=${UNIQUE_ID}"
                                        }

                                        // Capture JAR path
                                        def jar = sh(
                                            script: "ls target/*.jar | head -1",
                                            returnStdout: true
                                        ).trim()

                                        JAR_OUTPUT += "• ${svc.name}: ${jar}\n"
                                    }

                                    if (svc.type == "gradle") {
                                        sh "./gradlew clean build --quiet -PpipelineId=${UNIQUE_ID}"

                                        // Capture JAR path
                                        def jar = sh(
                                            script: "ls */build/libs/*.jar */*/build/libs/*.jar 2>/dev/null | head -1",
                                            returnStdout: true
                                        ).trim()

                                        JAR_OUTPUT += "• ${svc.name}: ${jar}\n"
                                    }

                                    echo "✔ BUILD COMPLETE: ${svc.name}\n"
                                }
                            }
                        }

                    }

                    parallel tasks
                }
            }
        }

        /* ---------------------------------------------------
         * CLEAN TEST EXECUTION (SHOWN ONLY WHEN FAILURE)
         * --------------------------------------------------- */
        stage('Test Services') {
            steps {
                script {
                    def tasks = [:]

                    SERVICES.each { svc ->

                        tasks[svc.name + "-tests"] = {
                            node {
                                dir(svc.path) {
                                    echo "\n▶ TEST START: ${svc.name}\n"

                                    try {
                                        if (svc.type == "maven") {
                                            withMaven(maven: 'Maven-3.9.11') {
                                                sh "mvn -q test -Dpipeline.id=${UNIQUE_ID}"
                                            }
                                        }

                                        if (svc.type == "gradle") {
                                            sh "./gradlew test --quiet -PpipelineId=${UNIQUE_ID}"
                                        }

                                        echo "✔ TEST PASSED: ${svc.name}\n"

                                    } catch (err) {
                                        echo "✘ TEST FAILED: ${svc.name}"
                                        echo "Showing logs (quiet mode suppressed earlier logs):"
                                        sh "cat build/test-results/test/*.xml || true"
                                        throw err
                                    }
                                }
                            }
                        }

                    }

                    parallel tasks
                }
            }
        }

        /* ---------------------------------------------------
         * FINAL SUMMARY BLOCK
         * --------------------------------------------------- */
        stage('Summary') {
            steps {
                script {
                    echo """
==================== PIPELINE SUMMARY ====================

Run ID: ${UNIQUE_ID}

BUILT ARTIFACTS:
${JAR_OUTPUT}

============================================================
"""
                }
            }
        }
    }
}
