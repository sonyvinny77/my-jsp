pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarServer') {
                    sh 'mvn clean verify sonar:sonar'
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
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
                sh 'echo Branch is: $BRANCH_NAME'   

                withCredentials([string(credentialsId: 'github-creds', variable: 'GITHUB_TOKEN')]) {
                    sh """
                        curl -X POST https://api.github.com/repos/jeevana1409/my-jsp/pulls \
                        -H "Authorization: token \$GITHUB_TOKEN" \
                        -H "Accept: application/vnd.github.v3+json" \
                        -d '{
                            "title": "Auto PR: ${env.GIT_BRANCH} → dev",
                            "head": "feature-branch-xyz",
                            "base": "dev",
                            "body": "Automatically created after successful SonarQube Quality Gate."
                        }'
                    """
                }
            }
        }

    }
}

