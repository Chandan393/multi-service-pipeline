pipeline {
    agent any

    stages {
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

        stage('Run Services in Parallel') {
            steps {
                script {
                    def tasks = [:]

                    SERVICES.each { svc ->
                        tasks[svc.name] = {
                            node {
                                echo "Processing ${svc.name}"

                                dir(svc.path) {

                                    if (svc.type == "gradle") {
                                        sh "./gradlew clean build -PpipelineId=${UNIQUE_ID}"
                                    }

                                    if (svc.type == "maven") {
                                        sh "mvn clean package -Dpipeline.id=${UNIQUE_ID}"
                                    }

                                    sh "echo 'Finished building ${svc.name}'"
                                }
                            }
                        }
                    }

                    parallel tasks
                }
            }
        }
    }
}

