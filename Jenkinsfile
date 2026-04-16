pipeline {
    agent any

    tools {
        maven 'maven'
    }

    parameters {
        string(name: 'APP_VERSION', description: 'Version from dev (example: v1.0.1)')
    }

    environment {
        DOCKER_IMAGE = "sony9014/mydeploy"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Use Version from Dev') {
            steps {
                script {
                    if (!params.APP_VERSION) {
                        error "APP_VERSION is required!"
                    }
                    env.APP_VERSION = params.APP_VERSION
                    echo "Using Version: ${env.APP_VERSION}"
                }
            }
        }

        stage('Update Maven Version') {
            steps {
                sh "mvn versions:set -DnewVersion=${APP_VERSION}"
            }
        }

        stage('Build WAR') {
            steps {
                sh "mvn clean package -DskipTests"
            }
        }

        stage('Run Tests') {
            steps {
                sh "mvn test"
            }
        }

        stage('Security Scan') {
            steps {
                sh "trivy fs --severity HIGH,CRITICAL --exit-code 1 ."
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {

                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin

                            docker build -t ${DOCKER_IMAGE}:${APP_VERSION} .

                            docker push ${DOCKER_IMAGE}:${APP_VERSION}

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
        stage('Manual Approval for PreProd') {
    steps {
        timeout(time: 10, unit: 'MINUTES') {
            input(
                message: 'Approve deployment to PreProd?',
                ok: 'Deploy',
                submitter: 'sony'
            )
        }
    }
}        
        stage('Trigger Deployment Repo - PreProd') {
    steps {
        script {
            build job: 'deployment-repo/preprod',
            parameters: [
                string(name: 'APP_VERSION', value: "${APP_VERSION}")
            ]
        }
    }
}
    }

    post {
        success {
            echo "✅ Release Pipeline Completed → QA Deployment Triggered"
        }
        failure {
            echo "❌ Release Pipeline Failed"
        }
    }
}
