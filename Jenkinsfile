pipeline {
agent any

tools {
    maven 'Maven'
}

environment {
    DOCKER_IMAGE = "sony9014/myapp"
    APP_VERSION = "${BUILD_NUMBER}"
}

stages {

    stage('Checkout') {
        steps {
            checkout scm
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

    stage('Deploy to QA') {
        steps {
            script {

                sshagent(credentials: ['docker-server-ssh']) {

                    sh """
                    ssh -o StrictHostKeyChecking=no ec2-user@3.16.188.115 "
                    docker pull ${DOCKER_IMAGE}:${APP_VERSION} &&
                    docker stop app || true &&
                    docker rm app || true &&
                    docker run -d -p 8080:8080 --name app-qa ${DOCKER_IMAGE}:${APP_VERSION}
                    "
                    """
                }
            }
        }
    }
}

post {
    success {
        echo "QA Deployment Successful"
    }

    failure {
        echo "QA Pipeline Failed"
    }
}
```

}

