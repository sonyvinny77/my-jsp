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
            def version = sh(
                script: "git describe --tags \$(git rev-list --tags --max-count=1)",
                returnStdout: true
            ).trim()

            echo "Latest Tag: ${version}"

            def newVersion = version.replace("v", "").tokenize('.')
            def major = newVersion[0]
            def minor = newVersion[1]
            def patch = newVersion[2].toInteger() + 1

            env.APP_VERSION = "v${major}.${minor}.${patch}"

            echo "Final Version: ${env.APP_VERSION}"

            sh """
                git config user.name jenkins
                git config user.email jenkins@local
                git tag ${env.APP_VERSION}
                git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/sonyvinny77/application-repo.git ${env.APP_VERSION}
            """
        }
    }
}
        stage('Update Maven Version') {
    steps {
        sh "mvn versions:set -DnewVersion=${env.VERSION}"
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
        sh 'mvn clean deploy -DskipTests'
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
