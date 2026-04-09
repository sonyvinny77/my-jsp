pipeline {
    agent any

    parameters {
        string(name: 'VERSION', description: 'Release version to deploy')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Version') {
            steps {
                script {
                    def version = params.VERSION

                    if (!version) {
                        error "❌ VERSION parameter is missing!"
                    }

                    if (version.contains("SNAPSHOT")) {
                        error "❌ SNAPSHOT version not allowed in MASTER!"
                    }

                    echo "✅ Valid Release Version: ${version}"
                    env.RELEASE_VERSION = version
                }
            }
        }

        stage('Trigger Deployment Repo (DEV)') {
            steps {
                build job: 'deployment-repo/dev', wait: false, parameters: [
                    string(name: 'VERSION', value: "${RELEASE_VERSION}")
                ]
            }
        }
    }

    post {
        success {
            echo "✅ Master triggered deployment pipeline"
        }
        failure {
            echo "❌ Master pipeline failed"
        }
    }
}
