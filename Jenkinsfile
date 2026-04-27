pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "dltlgud/msa-gatewayservice"
        CONTAINER_NAME = "msa-gatewayservice"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Gradle Build') {
            steps {
                sh 'chmod +x gradlew'
                sh './gradlew clean build -x test'
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub_info',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker build --provenance=false -t $DOCKER_IMAGE:latest -t $DOCKER_IMAGE:$BUILD_NUMBER .
                        docker push $DOCKER_IMAGE:latest
                        docker push $DOCKER_IMAGE:$BUILD_NUMBER
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker stop $CONTAINER_NAME || true
                    docker rm $CONTAINER_NAME || true
                    docker pull $DOCKER_IMAGE:latest
                    docker run -d --restart always \
                        --name $CONTAINER_NAME \
                        --network host \
                        -v /home/ubuntu/config/gateway/application.yml:/app/config/application.yml \
                        $DOCKER_IMAGE:latest
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
        success {
            sh """
                curl -s -X POST -H 'Content-type: application/json' \
                --data '{"text":"✅ *${env.JOB_NAME}* #${env.BUILD_NUMBER} 배포 성공"}' \
                ${env.SLACK_WEBHOOK_URL}
            """
        }
        failure {
            sh """
                curl -s -X POST -H 'Content-type: application/json' \
                --data '{"text":"❌ *${env.JOB_NAME}* #${env.BUILD_NUMBER} 배포 실패"}' \
                ${env.SLACK_WEBHOOK_URL}
            """
        }
    }
}
