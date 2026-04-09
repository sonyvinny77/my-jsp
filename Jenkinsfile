pipeline {
    agent any

    environment {
        NEXUS_URL = "http://172.31.42.87:8081"
        GROUP_ID = "com.example.maven-project"
        ARTIFACT_ID = "webapp"
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
                        error "❌ Not a SNAPSHOT version!"
                    }

                    env.SNAPSHOT_VERSION = version
                    env.RELEASE_VERSION = version.replace("-SNAPSHOT", "")

                    echo "Snapshot: ${SNAPSHOT_VERSION}"
                    echo "Release: ${RELEASE_VERSION}"
                }
            }
        }

        stage('Download Artifact from SNAPSHOT Repo') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {

                    sh '''
                    curl -u $NEXUS_USER:$NEXUS_PASS -o ${ARTIFACT_ID}.war \
                    "$NEXUS_URL/service/rest/v1/search/assets/download?repository=maven-dev&group=${GROUP_ID}&name=${ARTIFACT_ID}&version=${SNAPSHOT_VERSION}"

                    curl -u $NEXUS_USER:$NEXUS_PASS -o ${ARTIFACT_ID}.pom \
                    "$NEXUS_URL/service/rest/v1/search/assets/download?repository=maven-dev&group=${GROUP_ID}&name=${ARTIFACT_ID}&version=${SNAPSHOT_VERSION}&extension=pom"
                    '''
                }
            }
        }

        stage('Upload to RELEASE Repo') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {

                    sh '''
                    GROUP_PATH=$(echo $GROUP_ID | tr '.' '/')

                    curl -u $NEXUS_USER:$NEXUS_PASS --upload-file ${ARTIFACT_ID}.war \
                    $NEXUS_URL/repository/maven-releases/$GROUP_PATH/$ARTIFACT_ID/$RELEASE_VERSION/${ARTIFACT_ID}-${RELEASE_VERSION}.war

                    curl -u $NEXUS_USER:$NEXUS_PASS --upload-file ${ARTIFACT_ID}.pom \
                    $NEXUS_URL/repository/maven-releases/$GROUP_PATH/$ARTIFACT_ID/$RELEASE_VERSION/${ARTIFACT_ID}-${RELEASE_VERSION}.pom
                    '''
                }
            }
        }

        stage('Git Tag Release') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-cred',
                    usernameVariable: 'GIT_USERNAME',
                    passwordVariable: 'GIT_PASSWORD'
                )]) {

                    sh '''
                    git config user.name "jenkins"
                    git config user.email "jenkins@local"
                    git tag ${RELEASE_VERSION}
                    git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/sonyvinny77/application-repo.git ${RELEASE_VERSION}
                    '''
                }
            }
        }

        stage('Trigger Master Pipeline') {
            steps {
                build job: 'application-repo/master', wait: false, parameters: [
                    string(name: 'VERSION', value: "${RELEASE_VERSION}")
                ]
            }
        }
    }

    post {
        success {
            echo "✅ Artifact Promoted Successfully (NO REBUILD)"
        }
        failure {
            echo "❌ Promotion Failed"
        }
    }
}
