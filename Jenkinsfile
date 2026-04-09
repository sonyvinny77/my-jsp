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

        stage('Build & SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh 'mvn clean verify sonar:sonar'
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Create Pull Request to Dev') {
            when {
                not {
                    branch 'dev'
                }
            }
            steps {
                script {
                    def branchName = env.BRANCH_NAME
                    echo "Creating PR from ${branchName} to dev"

                    withCredentials([string(credentialsId: 'github-api-creds', variable: 'GITHUB_TOKEN')]) {
                        sh """
                        curl -X POST https://api.github.com/repos/sonyvinny77/application-repo/pulls \
                        -H "Authorization: token \$GITHUB_TOKEN" \
                        -H "Accept: application/vnd.github+json" \
                        -d '{
                          "title": "Auto PR: ${branchName} → dev",
                          "head": "${branchName}",
                          "base": "dev",
                          "body": "Created automatically by Jenkins."
                        }'
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
