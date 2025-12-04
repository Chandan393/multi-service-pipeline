pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        UNIQUE_ID = ""
        JAR_OUTPUT = ""
    }

    stages {

        stage('Init') {
            steps {
                script {
                    UNIQUE_ID = java.util.UUID.randomUUID().toString()
                }
            }
        }

        stage('Load Config') {
            steps {
                script {
                    def config = readYaml file: 'services.yaml'
                    SERVICES = config.services

                    echo """
===== PIPELINE INITIALIZED =====
Services Loaded : ${SERVICES.size()}
Run ID          : ${UNIQUE_ID}
================================
"""
                }
            }
        }

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

                                        def jar = sh(
                                            script: "ls target/*.jar | head -1",
                                            returnStdout: true
                                        ).trim()

                                        JAR_OUTPUT += "• ${svc.name}: ${jar}\n"
                                    }

                                    if (svc.type == "gradle") {
                                        sh "./gradlew clean build --quiet -PpipelineId=${UNIQUE_ID}"

                                        def jar = sh(
                                            script: "find . -type f -name '*.jar' | grep build/libs | head -1",
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
                                        echo "Showing failures only:"
                                        sh "grep -R \"<failure\" -n build/test-results/test || true"
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
