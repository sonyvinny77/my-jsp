pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        DOCKER_IMAGE = "sony9014/mydeploy"
        NEXUS_URL = "http://3.23.132.234:8081" // Updated to match your POM
        APP_REPO = "https://github.com/sonyvinny77/application-repo.git"
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

                    def latestTag = sh(
                        script: "git tag | grep '^v1\\.' | sort -V | tail -n 1 || echo v1.0.0",
                        returnStdout: true
                    ).trim()

                    echo "Filtered Latest Tag: ${latestTag}"

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

                        if (!tagExists) {
                            sh """
                            git tag ${APP_VERSION}
                            git push https://${GIT_USERNAME}:${GIT_PASSWORD}@${APP_REPO} ${APP_VERSION}
                            """
                        } else {
                            echo "⚠️ Tag already exists. Skipping..."
                        }
                    }
                }
            }
        }

        stage('Update Maven Version') {
            steps {
                sh "mvn -N versions:set -DnewVersion=${APP_VERSION}"
            }
        }

        stage('Build Multi-Module Project') {
            steps {
                // Build both server and webapp modules
                configFileProvider([configFile(fileId: 'maven-settings', variable: 'MAVEN_SETTINGS')]) {
                    sh "mvn clean install -s $MAVEN_SETTINGS -DskipTests"
                }
            }
        }

        stage('Run Tests') {
            steps {
                sh "mvn test"
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    configFileProvider([configFile(fileId: 'maven-settings', variable: 'MAVEN_SETTINGS')]) {
                        sh """
                        mvn deploy -s $MAVEN_SETTINGS -DskipTests \
                        -Dnexus.username=${NEXUS_USER} -Dnexus.password=${NEXUS_PASS}
                        """
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
