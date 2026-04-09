pipeline {
    agent any

    tools {
        maven 'maven'
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

        stage('Auto Version Increment') {
            steps {
                script {

                    sh 'git fetch --tags'

                    def latestTag = sh(
                        script: "git describe --tags \$(git rev-list --tags --max-count=1) 2>/dev/null || echo v1.0.0",
                        returnStdout: true
                    ).trim()

                    echo "Latest Tag: ${latestTag}"

                    def version = latestTag.replace("v","").tokenize('.')
                    def major = version[0]
                    def minor = version[1]
                    def patch = version[2].toInteger() + 1

                    env.APP_VERSION = "v${major}.${minor}.${patch}"

                    echo "New Version: ${APP_VERSION}"

                    withCredentials([usernamePassword(
                        credentialsId: 'github-cred',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )]) {

                        sh """
                        git config user.name "jenkins"
                        git config user.email "jenkins@local"

                        git tag ${APP_VERSION}

                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/sonyvinny77/application-repo.git ${APP_VERSION}
                        """
                    }
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

        stage('Upload Artifact to Nexus') {
            steps {
                sh "mvn deploy -DskipTests"
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

        stage('Trigger Deployment Repo - Dev') {
            steps {
                script {
                    build job: 'deployment-dev-pipeline',
                    parameters: [
                        string(name: 'APP_VERSION', value: "${APP_VERSION}")
                    ]
                }
            }
        }
    }

    post {
        success {
            echo "✅ Dev Pipeline Completed Successfully"
        }

        failure {
            echo "❌ Dev Pipeline Failed"
        }
    }
}
