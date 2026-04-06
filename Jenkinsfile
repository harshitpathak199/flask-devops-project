pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO = '210602314805.dkr.ecr.ap-south-1.amazonaws.com/flask-devops-app'
        CLUSTER = 'flask-eks-cluster'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/harshitpathak199/flask-devops-project.git'
                echo "Code cloned successfully"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image with tag: ${IMAGE_TAG}"
                    docker build -t flask-app:${IMAGE_TAG} .
                    echo "Docker image built successfully"
                '''
            }
        }

        stage('Push to AWS ECR') {
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh '''
                        echo "Logging into ECR..."
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REPO}

                        echo "Tagging image for ECR..."
                        docker tag flask-app:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}
                        docker tag flask-app:${IMAGE_TAG} ${ECR_REPO}:latest

                        echo "Pushing to ECR..."
                        docker push ${ECR_REPO}:${IMAGE_TAG}
                        docker push ${ECR_REPO}:latest
                        echo "Image pushed successfully"
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh '''
                        echo "Updating kubeconfig for EKS..."
                        aws eks update-kubeconfig \
                            --region ${AWS_REGION} \
                            --name ${CLUSTER}

                        echo "Updating image in deployment file..."
                        sed -i "s|IMAGE_PLACEHOLDER|${ECR_REPO}:${IMAGE_TAG}|g" k8s/deployment.yaml

                        echo "Applying Kubernetes manifests..."
                        kubectl apply -f k8s/

                        echo "Waiting for deployment to complete..."
                        kubectl rollout status deployment/flask-app

                        echo "Deployment successful!"
                        kubectl get pods
                        kubectl get services
                    '''
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh '''
                    docker rmi flask-app:${IMAGE_TAG} || true
                    docker rmi ${ECR_REPO}:${IMAGE_TAG} || true
                    echo "Cleanup done"
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS — Build ${BUILD_NUMBER} deployed!"
        }
        failure {
            echo "Pipeline FAILED — Check logs above for error"
        }
        always {
            echo "Pipeline finished — Build ${BUILD_NUMBER}"
        }
    }
}