pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "sony9014/mydeploy"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Get Latest Docker Version') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {

                        // Fetch ONLY version (no docker login noise)
                        env.APP_VERSION = sh(
                            script: """
                                curl -s -u \$DOCKER_USER:\$DOCKER_PASS https://hub.docker.com/v2/repositories/${DOCKER_IMAGE}/tags?page_size=100 | \
                                jq -r '.results[].name' | sort -V | tail -n1
                            """,
                            returnStdout: true
                        ).trim()
                    }

                    echo "Promoting Docker Version: ${env.APP_VERSION}"

                    if (!env.APP_VERSION) {
                        error "No Docker version found!"
                    }
                }
            }
        }

        stage('Trigger QA Deployment') {
            steps {
                script {
                    build job: 'deployment-repo/qa',
                    propagate: true,   // change to false if you don’t want failure to affect this pipeline
                    parameters: [
                        string(name: 'APP_VERSION', value: "${env.APP_VERSION}")
                    ]
                }
            }
        }
    }

    post {
        success {
            echo "✅ Release Pipeline Completed - QA Triggered"
        }
        failure {
            echo "❌ Release Pipeline Failed"
        }
    }
}
