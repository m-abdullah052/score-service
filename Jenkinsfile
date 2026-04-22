pipeline {
    agent any

    tools {
        jdk 'Java-17'
        maven 'Maven-3.8'
    }

    environment {
        AWS_REGION      = 'us-east-1'
        ECR_REGISTRY    = '123456789.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPO        = 'score-service'
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Prepare') {
            steps {
                checkout scm
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS \
                    --password-stdin ${ECR_REGISTRY}
                """
            }
        }

        stage('Build & Push Image') {
            steps {
                sh 'mvn clean package -DskipTests'
                sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
                sh """
                    docker tag ${ECR_REPO}:${IMAGE_TAG} \
                    ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}

                    docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('ECR Cleanup') {
            steps {
                sh """
                    aws ecr list-images \
                    --repository-name ${ECR_REPO} \
                    --filter tagStatus=UNTAGGED \
                    --query 'imageIds[*]' \
                    --output json | \
                    xargs -I {} aws ecr batch-delete-image \
                    --repository-name ${ECR_REPO} \
                    --image-ids {} || true
                """
            }
        }

        stage('Dev Deployment') {
            steps {
                sh "aws ecs update-service --cluster dev-cluster --service ${ECR_REPO}-dev --force-new-deployment --region ${AWS_REGION}"
            }
        }

        stage('Sandbox Deployment') {
            when { branch 'main' }
            steps {
                sh "aws ecs update-service --cluster sandbox-cluster --service ${ECR_REPO}-sandbox --force-new-deployment --region ${AWS_REGION}"
            }
        }

        stage('Preprod Deployment') {
            when { branch 'main' }
            steps {
                sh "aws ecs update-service --cluster preprod-cluster --service ${ECR_REPO}-preprod --force-new-deployment --region ${AWS_REGION}"
            }
        }

        stage('Production Deployment') {
            when { branch 'main' }
            input {
                message "Production par deploy karein?"
                ok "Deploy!"
            }
            steps {
                sh "aws ecs update-service --cluster prod-cluster --service ${ECR_REPO}-prod --force-new-deployment --region ${AWS_REGION}"
            }
        }
    }

    post {
        success { echo '✅ Pipeline successful!' }
        failure { echo '❌ Pipeline failed!' }
        always {
            sh "docker rmi ${ECR_REPO}:${IMAGE_TAG} || true"
        }
    }
}
