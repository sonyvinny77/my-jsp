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

        stage('Determine Latest Version') {
            steps {
                script {
                    // Get latest git tag
                    def version = sh(
                        script: "git describe --tags \$(git rev-list --tags --max-count=1)",
                        returnStdout: true
                    ).trim()

                    if (!version) {
                        error "No tags found! Please make sure Dev build created a tag."
                    }

                    env.APP_VERSION = version
                    echo "Promoting Version: ${env.APP_VERSION}"
                }
            }
        }

        stage('Verify Docker Image Exists') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker pull ${DOCKER_IMAGE}:${APP_VERSION}
                            docker logout
                        """
                    }
                }
            }
        }

        stage('Trigger Deployment Repo - QA') {
            steps {
                script {
                    build job: 'deployment-repo/qa',
                    parameters: [
                        string(name: 'APP_VERSION', value: "${APP_VERSION}")
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
