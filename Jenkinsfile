pipeline {
    agent any

    // ----------------------------------------------------------
    // GLOBAL PIPELINE VARIABLES (Persist Between Stages)
    // ----------------------------------------------------------
    environment {
        UNIQUE_ID = "${UUID.randomUUID().toString()}"
    }

    // Groovy shared vars (must be declared outside stages)
    options {
        timestamps()
    }

    // Shared objects
    stages {

        // Declare Groovy variables outside stages
        stage('Initialize') {
            steps {
                script {
                    // Global shared variables
                    SERVICES = []
                    JAR_OUTPUT = [:]   // Map: service -> jar path

                    echo """
===================== PIPELINE START =====================
Run ID      : ${UNIQUE_ID}
==========================================================
"""
                }
            }
        }

        // ----------------------------------------------------------
        // LOAD YAML SERVICE CONFIG
        // ----------------------------------------------------------
        stage('Load Config') {
            steps {
                script {
                    def config = readYaml(file: 'services.yaml')
                    SERVICES = config.services

                    echo """
Loaded Services : ${SERVICES.size()}
Services: ${SERVICES*.name.join(', ')}
"""
                }
            }
        }

        // ----------------------------------------------------------
        // PARALLEL BUILD (QUIET MODE)
        // ----------------------------------------------------------
        stage('Build Services') {
            steps {
                script {

                    def buildTasks = SERVICES.collectEntries { svc ->

                        ["BUILD-${svc.name}": {

                            node {
                                dir(svc.path) {

                                    echo "▶ Building ${svc.name}"

                                    if (svc.type == "maven") {
                                        withMaven(maven: 'Maven-3.9.11') {
                                            sh "mvn -q clean package -Dpipeline.id=${UNIQUE_ID}"
                                        }

                                        def jar = sh(
                                            script: "ls target/*.jar | head -1",
                                            returnStdout: true
                                        ).trim()

                                        JAR_OUTPUT[svc.name] = jar
                                    }

                                    if (svc.type == "gradle") {
                                        sh "./gradlew clean build --quiet -PpipelineId=${UNIQUE_ID}"

                                        def jar = sh(
                                            script: "find build -type f -name '*.jar' | head -1",
                                            returnStdout: true
                                        ).trim()

                                        JAR_OUTPUT[svc.name] = jar
                                    }

                                    echo "✔ Build complete: ${svc.name}"
                                }
                            }
                        }]
                    }

                    parallel buildTasks
                }
            }
        }

        // ----------------------------------------------------------
        // PARALLEL TEST (QUIET MODE, SHOW LOGS ON FAILURE)
        // ----------------------------------------------------------
        stage('Test Services') {
            steps {
                script {

                    def testTasks = SERVICES.collectEntries { svc ->

                        ["TEST-${svc.name}": {

                            node {
                                dir(svc.path) {
                                    echo "▶ Testing ${svc.name}"

                                    try {
                                        if (svc.type == "maven") {
                                            withMaven(maven: 'Maven-3.9.11') {
                                                sh "mvn -q test -Dpipeline.id=${UNIQUE_ID}"
                                            }
                                        }

                                        if (svc.type == "gradle") {
                                            sh "./gradlew test --quiet -PpipelineId=${UNIQUE_ID}"
                                        }

                                        echo "✔ Tests passed: ${svc.name}"

                                    } catch (ex) {
                                        echo "✘ Tests FAILED for ${svc.name}"
                                        echo "Showing test failure details…"

                                        sh """
                                            echo "----- FAILED TEST OUTPUT -----"
                                            find build/test-results -name '*.xml' -print -exec cat {} \\; || true
                                        """

                                        throw ex
                                    }
                                }
                            }
                        }]
                    }

                    parallel testTasks
                }
            }
        }

        // ----------------------------------------------------------
        // FINAL SUMMARY (CLEAN OUTPUT)
        // ----------------------------------------------------------
        stage('Summary') {
            steps {
                script {

                    def summary = """
====================== BUILD SUMMARY ======================

Pipeline Run ID : ${UNIQUE_ID}

Generated Artifacts:
------------------------------------------------------------
"""

                    JAR_OUTPUT.each { svc, jar ->
                        summary += "• ${svc.padRight(15)} → ${jar}\n"
                    }

                    summary += """
============================================================
"""

                    echo summary
                }
            }
        }
    }
}
