pipeline {
    agent any

    tools {
        maven 'maven'
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
        stage('Auto Version Increment') {
    steps {
        script {

            sh '''
            git config --global --add safe.directory '*'

            # Fetch latest tags from remote
            git fetch --tags
            '''

            // Get latest tag properly (sorted)
            def latestTag = sh(
                script: "git tag --sort=-v:refname | head -n 1 || echo v1.0.0",
                returnStdout: true
            ).trim()

            echo "Latest Tag: ${latestTag}"

            // Extract version
            def version = latestTag.replace("v","").tokenize('.')
            def major = version[0]
            def minor = version[1]
            def patch = version[2].toInteger() + 1

            env.APP_VERSION = "v${major}.${minor}.${patch}"

            echo "New Version: ${env.APP_VERSION}"

            withCredentials([usernamePassword(
                credentialsId: 'github-cred',
                usernameVariable: 'GIT_USERNAME',
                passwordVariable: 'GIT_PASSWORD'
            )]) {

                sh '''
                git config user.name "jenkins"
                git config user.email "jenkins@local"
                '''

                // ✅ Check if tag already exists
                def tagExists = sh(
                    script: "git tag -l ${env.APP_VERSION}",
                    returnStdout: true
                ).trim()

                if (tagExists) {
                    echo "⚠️ Tag already exists. Skipping tag creation..."
                } else {
                    sh """
                    git tag ${APP_VERSION}
                    git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/sonyvinny77/my-jsp.git ${APP_VERSION}
                    """
                }
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

        stage('Deploy to Dev') {
            steps {
                script {

                    sshagent(credentials: ['docker-server-ssh']) {

                        sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@172.31.12.153 "
                        docker pull ${DOCKER_IMAGE}:${APP_VERSION} &&
                        docker stop app || true &&
                        docker rm app || true &&
                        docker run -d -p 8080:8080 --name app ${DOCKER_IMAGE}:${APP_VERSION}
                        "
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline Completed Successfully"
        }

        failure {
            echo "❌ Pipeline Failed"
        }
    }
}
