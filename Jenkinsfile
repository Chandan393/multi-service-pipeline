def SERVICES = []
def JAR_PATHS = [:]

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

        stage('Build Services') {
            steps {
                script {

                    JAR_PATHS = [:]  // reset

                    def branches = SERVICES.collectEntries { svc ->
                        ["BUILD-${svc.name}": {
                            node {
                                dir(svc.path) {

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
                expression { currentBuild.resultIsBetterOrEqualTo('SUCCESS') }
            }
            steps {
                script {
                    def branches = SERVICES.collectEntries { svc ->
                        ["TEST-${svc.name}": {
                            node {
                                dir(svc.path) {
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

        /* -------------------------------------------------------
         *  NEW PARALLEL DEPLOYMENT STAGE (ADDED)
         * ------------------------------------------------------- */
        stage('Deploy Services') {
            when {
                expression { currentBuild.resultIsBetterOrEqualTo('SUCCESS') }
            }
            steps {
                script {
                    def branches = SERVICES.collectEntries { svc ->
                        ["DEPLOY-${svc.name}": {
                            node {

                                echo "🚀 Deploying ${svc.name}"
                                echo "Using JAR: ${JAR_PATHS[svc.name]}"

                                // Replace this with your real deploy command
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
                expression { currentBuild.resultIsBetterOrEqualTo('SUCCESS') }
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
