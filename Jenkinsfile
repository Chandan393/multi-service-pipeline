pipeline {
    agent any

    stages {

        /* -----------------------------
         * LOAD CONFIG
         * ----------------------------- */
        stage('Load Config') {
            steps {
                script {
                    def config = readYaml file: 'services.yaml'
                    SERVICES = config.services
                    UNIQUE_ID = UUID.randomUUID().toString()

                    echo "Loaded ${SERVICES.size()} services"
                    echo "Pipeline Run ID: ${UNIQUE_ID}"
                }
            }
        }

        /* -----------------------------
         * BUILD ALL SERVICES IN PARALLEL
         * CLEAN LOGS — NO -X / --info
         * ----------------------------- */
        stage('Build Services') {
            steps {
                script {
                    def buildTasks = [:]

                    SERVICES.each { svc ->
                        buildTasks[svc.name] = {
                            node {
                                echo "Building: ${svc.name}"

                                dir(svc.path) {

                                    if (svc.type == "maven") {
                                        withMaven(maven: 'Maven-3.9.11') {
                                            sh "mvn clean package -Dpipeline.id=${UNIQUE_ID}"
                                        }
                                    }

                                    if (svc.type == "gradle") {
                                        sh "./gradlew clean build -PpipelineId=${UNIQUE_ID}"
                                    }

                                    echo "Build Completed for ${svc.name}"
                                }
                            }
                        }
                    }

                    parallel buildTasks
                }
            }
        }

        /* -----------------------------
         * TEST ALL SERVICES IN PARALLEL
         * KEEP DETAILED LOGS
         * ----------------------------- */
        stage('Test Services') {
            steps {
                script {
                    def testTasks = [:]

                    SERVICES.each { svc ->
                        testTasks[svc.name + "-tests"] = {
                            node {
                                echo "Running Tests for: ${svc.name}"

                                dir(svc.path) {

                                    if (svc.type == "maven") {
                                        withMaven(maven: 'Maven-3.9.11') {
                                            sh "mvn test -Dpipeline.id=${UNIQUE_ID} -X"
                                        }
                                    }

                                    if (svc.type == "gradle") {
                                        sh "./gradlew test -PpipelineId=${UNIQUE_ID} --info"
                                    }

                                    echo "Tests Completed for ${svc.name}"
                                }
                            }
                        }
                    }

                    parallel testTasks
                }
            }
        }
    }
}
