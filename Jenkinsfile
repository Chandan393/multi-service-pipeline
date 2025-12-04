def SERVICES = []
def CHANGED_SERVICES = []
def JAR_PATHS = [:]

pipeline {
    agent any

    environment {
        RUN_ID = UUID.randomUUID().toString()
        COMMITS_DIR = "${WORKSPACE}/.last_commits"
    }

    stages {

        stage('Initialize') {
            steps {
                echo """
                ===================== PIPELINE START =====================
                Run ID      : ${RUN_ID}
                ==========================================================
                """
                sh "mkdir -p ${COMMITS_DIR}"
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

        stage('Detect Changes') {
            steps {
                script {
                    CHANGED_SERVICES = []

                    SERVICES.each { svc ->
                        echo "Checking changes for ${svc.name}"

                        // fresh clone
                        sh """
                            rm -rf tmp_${svc.name}
                            git clone --depth 1 -b ${svc.branch} ${svc.repo} tmp_${svc.name}
                        """

                        // latest commit ID
                        def latestCommit = sh(
                            script: "cd tmp_${svc.name} && git rev-parse HEAD",
                            returnStdout: true
                        ).trim()

                        def commitFile = "${COMMITS_DIR}/${svc.name}.txt"
                        def lastCommit = fileExists(commitFile) ? readFile(commitFile).trim() : ""

                        if (lastCommit != latestCommit) {
                            echo "✔ Changes detected in ${svc.name}"
                            CHANGED_SERVICES.add(svc)
                        } else {
                            echo "No changes in ${svc.name}"
                        }

                        // Save new commit
                        writeFile(file: commitFile, text: latestCommit)
                    }

                    echo "Changed Services: ${CHANGED_SERVICES*.name}"

                    // STOP PIPELINE IF NO CHANGES
                    if (CHANGED_SERVICES.isEmpty()) {
                        echo "No service changed → Skipping build/test/deploy → SUCCESS"
                        currentBuild.result = "SUCCESS"
                        return
                    }
                }
            }
        }

        stage('Build Services') {
            when {
                expression { CHANGED_SERVICES.size() > 0 }
            }
            steps {
                script {

                    JAR_PATHS = [:]  // reset

                    def branches = CHANGED_SERVICES.collectEntries { svc ->
                        ["BUILD-${svc.name}": {
                            node {
                                dir("build_${svc.name}") {

                                    // fresh clone for actual build
                                    sh "rm -rf build_${svc.name}"
                                    sh "git clone -b ${svc.branch} ${svc.repo} build_${svc.name}"

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
