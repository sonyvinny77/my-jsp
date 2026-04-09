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

        stage('Get Snapshot Version') {
            steps {
                script {
                    def version = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()

                    echo "Current Version: ${version}"

                    if (!version.contains("SNAPSHOT")) {
                        error "❌ Not a SNAPSHOT version. Release should come from SNAPSHOT!"
                    }

                    env.RELEASE_VERSION = version.replace("-SNAPSHOT", "")
                    echo "Release Version: ${env.RELEASE_VERSION}"
                }
            }
        }

        stage('Set Release Version') {
            steps {
                sh "mvn -N versions:set -DnewVersion=${RELEASE_VERSION}"
            }
        }

        stage('Build Artifact') {
            steps {
                sh "mvn clean install -DskipTests"
            }
        }

        stage('Run Tests') {
            steps {
                sh "mvn test"
            }
        }

        stage('Upload to Nexus (Release Repo)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    configFileProvider([configFile(fileId: 'maven-settings', variable: 'MAVEN_SETTINGS')]) {
                        sh """
                        mvn deploy -DskipTests -s $MAVEN_SETTINGS \
                        -Dnexus.username=${NEXUS_USER} \
                        -Dnexus.password=${NEXUS_PASS}
                        """
                    }
                }
            }
        }

        stage('Git Tag Release') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'github-cred',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )]) {
                        sh '''
                        git config user.name "jenkins"
                        git config user.email "jenkins@local"
                        '''

                        sh """
                        git tag ${RELEASE_VERSION}
                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/sonyvinny77/application-repo.git ${RELEASE_VERSION}
                        """
                    }
                }
            }
        }

        stage('Docker Build & Push (Release Image)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                    echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                    docker build -t ${DOCKER_IMAGE}:${RELEASE_VERSION} .
                    docker push ${DOCKER_IMAGE}:${RELEASE_VERSION}
                    docker logout
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Release Pipeline Successful"
        }
        failure {
            echo "❌ Release Pipeline Failed"
        }
    }
}
