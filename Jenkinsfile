pipeline {
    agent any

    stages {

        stage('Load Config') {
            steps {
                script {
                    def config = readYaml file: 'services.yaml'
                    SERVICES = config.services
                    UNIQUE_ID = UUID.randomUUID().toString()

                    echo "✔ Loaded ${SERVICES.size()} services"
                    echo "✔ Pipeline Run ID: ${UNIQUE_ID}"
                }
            }
        }

        /* ---------------------------------------
         * CLEAN BUILD (Quiet Mode)
         * --------------------------------------- */
        stage('Build Services') {
            steps {
                script {
                    def buildTasks = [:]

                    SERVICES.each { svc ->
                        buildTasks[svc.name] = {
                            node {

                                dir(svc.path) {
                                    echo "▶ Building: ${svc.name}"

                                    if (svc.type == "maven") {
                                        withMaven(maven: 'Maven-3.9.11') {
                                            sh "mvn -q clean package -Dpipeline.id=${UNIQUE_ID}"
                                        }
                                    }

                                    if (svc.type == "gradle") {
                                        sh "./gradlew clean build --quiet -PpipelineId=${UNIQUE_ID}"
                                    }

                                    echo "✔ Build completed: ${svc.name}"
                                }
                            }
                        }
                    }

                    parallel buildTasks
                }
            }
        }

        /* ---------------------------------------
         * CLEAN TEST EXECUTION (Quiet Mode)
         * --------------------------------------- */
        stage('Test Services') {
            steps {
                script {
                    def testTasks = [:]

                    SERVICES.each { svc ->
                        testTasks[svc.name + "-tests"] = {
                            node {

                                dir(svc.path) {
                                    echo "▶ Testing: ${svc.name}"

                                    if (svc.type == "maven") {
                                        withMaven(maven: 'Maven-3.9.11') {
                                            sh "mvn -q test -Dpipeline.id=${UNIQUE_ID}"
                                        }
                                    }

                                    if (svc.type == "gradle") {
                                        sh "./gradlew test --quiet -PpipelineId=${UNIQUE_ID}"
                                    }

                                    echo "✔ Tests completed: ${svc.name}"
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
