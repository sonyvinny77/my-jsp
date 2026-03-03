pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        DOCKER_IMAGE = "sony9014/myapp"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Initialization - Version Check') {
            steps {
                script {
                    echo "Fetching tags..."
                    sh "git fetch --tags"

                    env.APP_VERSION = sh(
                        script: "git describe --tags --abbrev=0",
                        returnStdout: true
                    ).trim()

                    echo "Using version: ${APP_VERSION}"
                }
            }
        }

        stage('Build') {
            steps {
                sh "mvn clean package"
            }
        }

        stage('Test') {
            steps {
                sh "mvn test"
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

        stage('Deploy to Dev Server') {
            steps {

                sh 'echo Branch is: $BRANCH_NAME'   

                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh """
                        curl -X POST https://api.github.com/repos/sonyvinny77/my-jsp/pulls \
                        -H "Authorization: token \$GITHUB_TOKEN" \
                        -H "Accept: application/vnd.github.v3+json" \
                        -d '{
                            "title": "Auto PR: ${env.GIT_BRANCH} → dev",
                            "head": "feature/login",
                            "base": "dev",
                            "body": "Automatically created after successful SonarQube Quality Gate."
                        }'
                    """
                script {
                    sshagent(credentials: ['docker-server-ssh']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@18.216.228.137 "
                            docker pull ${DOCKER_IMAGE}:${APP_VERSION} &&
                            docker stop app || true &&
                            docker rm app || true &&
                            docker run -d -p 8080:8080 --restart=always --name app ${DOCKER_IMAGE}:${APP_VERSION}
                        "
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ DEV Deployment Successful!"
        }
        failure {
            echo "❌ DEV Deployment Failed!"
        }
    }
}
