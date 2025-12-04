pipeline {
    agent any

    environment {
        RUN_ID = UUID.randomUUID().toString()
    }

    stages {

        stage('Initialize') {
            steps {
                script {
                    def SERVICES = []
                    def JAR_OUTPUT = [:]

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
                    def SERVICES = config.services

                    echo """
                    Loaded Services : ${SERVICES.size()}
                    Services: ${SERVICES.join(', ')}
                    """
                }
            }
        }

        stage('Build Services') {
            steps {
                script {
                    def config = readYaml file: 'services.yaml'
                    def SERVICES = config.services

                    def branches = SERVICES.collectEntries { service ->
                        ["BUILD-${service}": {
                            node {
                                dir("${service}") {
                                    echo "▶ Building ${service}"

                                    if (fileExists('pom.xml')) {
                                        withMaven() {
                                            sh "mvn -q clean package -Dpipeline.id=${RUN_ID}"
                                        }
                                    } else if (fileExists('build.gradle')) {
                                        sh "./gradlew clean build --quiet -PpipelineId=${RUN_ID}"
                                    }

                                    sh "find . -name '*.jar' | head -1"
                                    echo "✔ Build complete: ${service}"
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
                    def SERVICES = config.services

                    def branches = SERVICES.collectEntries { service ->
                        ["TEST-${service}": {
                            node {
                                dir("${service}") {
                                    echo "▶ Testing ${service}"

                                    if (fileExists('pom.xml')) {
                                        withMaven() {
                                            sh "mvn -q test -Dpipeline.id=${RUN_ID}"
                                        }
                                    } else if (fileExists('build.gradle')) {
                                        sh "./gradlew test --quiet -PpipelineId=${RUN_ID}"
                                    }

                                    echo "✔ Tests passed: ${service}"
                                }
                            }
                        }]
                    }

                    parallel branches
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
