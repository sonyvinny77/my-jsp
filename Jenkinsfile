pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        DOCKER_IMAGE = "sony9014/mydeploy"
        NEXUS_URL = "http://172.31.42.87:8081"
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
            git fetch --tags
            '''

            // ✅ Only consider tags starting from v1.x.x (fresh start)
            def latestTag = sh(
                script: "git tag | grep '^v1\\.' | sort -V | tail -n 1 || echo v1.0.0",
                returnStdout: true
            ).trim()

            echo "Filtered Latest Tag: ${latestTag}"

            // If no valid tag found → start from v1.0.0
            if (latestTag == "" || latestTag == "v1.0.0") {
                env.APP_VERSION = "v1.0.0"
            } else {
                def version = latestTag.replace("v","").tokenize('.')
                def major = version[0]
                def minor = version[1]
                def patch = version[2].toInteger() + 1

                env.APP_VERSION = "v${major}.${minor}.${patch}"
            }

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

                def tagExists = sh(
                    script: "git tag -l ${env.APP_VERSION}",
                    returnStdout: true
                ).trim()

                if (tagExists) {
                    echo "⚠️ Tag already exists. Skipping..."
                } else {
                    sh """
                    git tag ${APP_VERSION}
                    git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/sonyvinny77/application-repo.git ${APP_VERSION}
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
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {

                        configFileProvider([configFile(fileId: 'maven-settings', variable: 'MAVEN_SETTINGS')]) {

                            sh """
                            mvn deploy \
                            -DskipTests \
                            -s $MAVEN_SETTINGS \
                            -Dnexus.username=$NEXUS_USER \
                            -Dnexus.password=$NEXUS_PASS
                            """
                        }
                    }
                }
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
