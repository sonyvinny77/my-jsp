pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        DOCKER_IMAGE = "sony9014/myapp"
        NEXUS_URL = "http://18.117.196.20:8081"
        NEXUS_REPO = "my-maven-releases" // Your hosted Maven repo name
        NEXUS_CRED_ID = "nexus-creds"
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
                        script: "git describe --tags --abbrev=0 || echo v0.0.0",
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

                    // Git tagging
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

        stage('Upload WAR to Nexus') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: "${NEXUS_CRED_ID}",
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh """
                        mvn deploy:deploy-file \
                            -DgroupId=com.devops \
                            -DartifactId=myapp \
                            -Dversion=${APP_VERSION} \
                            -Dpackaging=war \
                            -Dfile=target/myapp.war \
                            -DrepositoryId=nexus \
                            -Durl=${NEXUS_URL}/repository/${NEXUS_REPO}/ \
                            -Dusername=${NEXUS_USER} \
                            -Dpassword=${NEXUS_PASS}
                        """
                    }
                }
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
                        ssh -o StrictHostKeyChecking=no ec2-user@3.140.198.236 "
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
                input message: "Approve deployment to QA?"
                script {
                    sshagent(credentials: ['docker-server-ssh']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@18.217.181.10 "
                            docker pull ${DOCKER_IMAGE}:${APP_VERSION} &&
                            docker stop app || true &&
                            docker rm app || true &&
                            docker run -d -p 8080:8080 --restart=always --name app-qa ${DOCKER_IMAGE}:${APP_VERSION}
                        "
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ DEV & QA Deployment Successful!"
        }
        failure {
            echo "❌ Deployment Failed!"
        }
    }
}
