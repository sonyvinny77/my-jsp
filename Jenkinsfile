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
            sh "git fetch --tags"

            def latestTag = sh(
                script: "git describe --tags --abbrev=0",
                returnStdout: true
            ).trim()

            echo "Latest Tag: ${latestTag}"

            def versionNumber = latestTag.replace("v", "").tokenize(".")
            def major = versionNumber[0]
            def minor = versionNumber[1]
            def patch = versionNumber[2].toInteger() + 1

            def newTag = "v${major}.${minor}.${patch}"
            echo "New Version: ${newTag}"

            env.APP_VERSION = newTag

            // Check if tag already exists
            def tagExists = sh(
                script: "git tag -l ${newTag}",
                returnStdout: true
            ).trim()

            if(tagExists) {
                echo "Tag already exists: ${newTag}"
            } else {

                sh "git tag ${newTag}"

                withCredentials([usernamePassword(
                    credentialsId: 'github-api-creds',
                    usernameVariable: 'GIT_USERNAME',
                    passwordVariable: 'GIT_PASSWORD'
                )]) {

                    sh """
                    git config user.name "${GIT_USERNAME}"
                    git config user.email "${GIT_USERNAME}@users.noreply.github.com"
                    git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/sonyvinny77/my-jsp.git ${newTag}
                    """
                }
            }
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
                        ssh -o StrictHostKeyChecking=no ec2-user@3.146.221.101 "
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
