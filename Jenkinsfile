pipeline {
    agent any

    environment {
        IMAGE_NAME = "sony9014/regapp"
        DEV_SERVER  = "ubuntu@44.206.233.83"
        PROD_SERVER = "ubuntu@44.201.186.253"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/sonyvinny77/maven.git'
            }
        }

        stage('Build Maven') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Create Dockerfile & Build Image') {
            steps {
                sh '''
cat <<EOF > Dockerfile
FROM tomcat:9-jdk11
COPY webapp/target/webapp.war /usr/local/tomcat/webapps/
EXPOSE 8080
EOF

docker build -t $IMAGE_NAME:$BUILD_NUMBER .
docker tag $IMAGE_NAME:$BUILD_NUMBER $IMAGE_NAME:latest
                '''
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                sh '''
docker push $IMAGE_NAME:$BUILD_NUMBER
docker push $IMAGE_NAME:latest
                '''
            }
        }

        stage('Deploy to DEV') {
            steps {
                sh """
ssh -o StrictHostKeyChecking=no $DEV_SERVER '
docker pull $IMAGE_NAME:latest
docker stop dev-container || true
docker rm dev-container || true
docker run -d -p 8081:8080 --name dev-container $IMAGE_NAME:latest
'
                """
            }
        }

        stage('Deploy to PROD') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                sh '''
ssh -o StrictHostKeyChecking=no ubuntu@44.201.186.253 << EOF
docker pull sony9014/regapp:latest
docker stop prod-container || true
docker rm prod-container || true
docker run -d -p 8082:8080 --name prod-container sony9014/regapp:latest
exit
EOF
'''

                }
            }
        }
    }
}

