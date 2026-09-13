pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        ECR_REGISTRY = "465708537536.dkr.ecr.ap-south-1.amazonaws.com"
        AWS_ACCESS_KEY_ID = credentials("jenkins-ecr-access-key-id")
        AWS_SECRET_ACCESS_KEY = credentials("jenkins-ecr-secret-access-key")
    }

    stages {
        stage("Checkout") {
            steps {
                checkout scm
            }
        }

        stage("ECR Login") {
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                """
            }
        }

        stage("Build & Push Images") {
            steps {
                sh """
                    docker build -t ${ECR_REGISTRY}/streaming-auth:${BUILD_NUMBER} backend/authService
                    docker push ${ECR_REGISTRY}/streaming-auth:${BUILD_NUMBER}

                    docker build -t ${ECR_REGISTRY}/streaming-stream:${BUILD_NUMBER} -f backend/streamingService/Dockerfile backend
                    docker push ${ECR_REGISTRY}/streaming-stream:${BUILD_NUMBER}

                    docker build -t ${ECR_REGISTRY}/streaming-admin:${BUILD_NUMBER} -f backend/adminService/Dockerfile backend
                    docker push ${ECR_REGISTRY}/streaming-admin:${BUILD_NUMBER}

                    docker build -t ${ECR_REGISTRY}/streaming-chat:${BUILD_NUMBER} -f backend/chatService/Dockerfile backend
                    docker push ${ECR_REGISTRY}/streaming-chat:${BUILD_NUMBER}

                    docker build -t ${ECR_REGISTRY}/streaming-frontend:${BUILD_NUMBER} frontend
                    docker push ${ECR_REGISTRY}/streaming-frontend:${BUILD_NUMBER}
                """
            }
        }
    }

    post {
        success {
            echo "All 5 images built and pushed to ECR with tag ${BUILD_NUMBER}"
            sh """
                aws sns publish --topic-arn arn:aws:sns:ap-south-1:465708537536:streamingapp-deployments --subject "Jenkins Build Success" --message "StreamingApp Jenkins build ${BUILD_NUMBER} succeeded - all 5 images built and pushed to ECR." --region ap-south-1
            """
        }
        failure {
            echo "Pipeline failed - check the stage logs above"
            sh """
                aws sns publish --topic-arn arn:aws:sns:ap-south-1:465708537536:streamingapp-deployments --subject "Jenkins Build Failed" --message "StreamingApp Jenkins build ${BUILD_NUMBER} failed - check the pipeline logs." --region ap-south-1
            """
        }
    }
}
