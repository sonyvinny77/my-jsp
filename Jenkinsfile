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

        stage('Initialization - Auto Version Increment') {
            steps {
                script {

                    sh 'git fetch --tags'

                    def latestTag = sh(
                        script: "git describe --tags \$(git rev-list --tags --max-count=1) || echo v1.0.0",
                        returnStdout: true
                    ).trim()

                    echo "Latest Tag: ${latestTag}"

                    def version = latestTag.replace("v","").tokenize('.')
                    def major = version[0]
                    def minor = version[1]
                    def patch = version[2].toInteger() + 1

                    env.APP_VERSION = "v${major}.${minor}.${patch}"

                    echo "New Version: ${env.APP_VERSION}"

                    withCredentials([usernamePassword(
                        credentialsId: 'github-api-creds',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )]) {

                        sh """
                        git config user.name "jenkins"
                        git config user.email "jenkins@local"

                        git tag ${APP_VERSION}

                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/sonyvinny77/my-jsp.git ${APP_VERSION}
                        """
                    }
                }
            }
        }

        stage('Build WAR') {
            steps {
                sh "mvn clean package"

                sh """
                cp webapp/target/webapp.war webapp/target/webapp-${APP_VERSION}.war
                """
            }
        }

        stage('Test') {
            steps {
                sh "mvn test"
            }
        }

        stage('Upload WAR to Nexus') {
            steps {
                sh "mvn deploy -DskipTests"
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                trivy fs --severity HIGH,CRITICAL --exit-code 1 .
                '''
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

                        docker build --no-cache -t ${DOCKER_IMAGE}:${APP_VERSION} .

                        docker push ${DOCKER_IMAGE}:${APP_VERSION}

                        docker logout
                        """
                    }
                }
            }
        }

        stage('Deploy to Dev Server') {
            steps {
                script {
                    sshagent(credentials: ['docker-server-ssh']) {

                        sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@18.191.31.74 "
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

        stage('Deploy to QA Server') {
            steps {

                input "Deploy to QA environment?"

                script {

                    sshagent(credentials: ['docker-server-ssh']) {

                        sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@3.140.199.196 "
                        docker pull ${DOCKER_IMAGE}:${APP_VERSION} &&
                        docker stop qa-app || true &&
                        docker rm qa-app || true &&
                        docker run -d -p 8080:8080 --name qa-app ${DOCKER_IMAGE}:${APP_VERSION}
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
