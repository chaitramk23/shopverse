pipeline {
    agent any

    environment {
        AWS_REGION       = 'ap-south-1'
        ECR_REGISTRY     = 'ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com'
        EKS_CLUSTER_NAME = 'shopverse-cluster'
        TF_STATE_BUCKET  = 'shopverse-tf-state'

        FRONTEND_IMAGE = "${ECR_REGISTRY}/shopverse-frontend:${BUILD_NUMBER}"
        BACKEND_IMAGE  = "${ECR_REGISTRY}/shopverse-backend:${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/your-org/shopverse.git'
            }
        }

        stage('Backend Tests') {
            steps {
                dir('backend') {
                    sh 'go mod tidy'
                    sh 'go test ./...'
                }
            }
        }

        stage('Frontend Lint') {
            steps {
                dir('frontend') {
                    sh 'npm install'
                    sh 'npx eslint . --ext js,jsx'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                docker build -t shopverse-frontend:scan ./frontend
                docker build -t shopverse-backend:scan ./backend
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                trivy image --exit-code 1 --severity CRITICAL \
                shopverse-frontend:scan

                trivy image --exit-code 1 --severity CRITICAL \
                shopverse-backend:scan
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION \
                | docker login --username AWS \
                --password-stdin $ECR_REGISTRY
                '''
            }
        }

        stage('Build & Tag Images') {
            steps {
                sh '''
                docker build -t $FRONTEND_IMAGE \
                -t $ECR_REGISTRY/shopverse-frontend:latest ./frontend

                docker build -t $BACKEND_IMAGE \
                -t $ECR_REGISTRY/shopverse-backend:latest ./backend
                '''
            }
        }

        stage('Push Images') {
            steps {
                sh '''
                docker push $FRONTEND_IMAGE
                docker push $ECR_REGISTRY/shopverse-frontend:latest

                docker push $BACKEND_IMAGE
                docker push $ECR_REGISTRY/shopverse-backend:latest
                '''
            }
        }

        stage('Package Helm Chart') {
            steps {
                sh '''
                helm package ./helm/shopverse \
                --version 1.0.${BUILD_NUMBER} \
                --app-version ${BUILD_NUMBER}
                '''
            }
        }

        stage('Push Helm Chart') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION | \
                helm registry login --username AWS \
                --password-stdin $ECR_REGISTRY

                helm push shopverse-*.tgz \
                oci://$ECR_REGISTRY/shopverse-helmchart
                '''
            }
        }

        stage('Terraform Provision') {
            steps {
                dir('terraform') {
                    sh '''
                    terraform init
                    terraform plan -out=tfplan
                    terraform apply -auto-approve tfplan
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                aws eks update-kubeconfig \
                --name $EKS_CLUSTER_NAME \
                --region $AWS_REGION

                helm upgrade --install shopverse \
                oci://$ECR_REGISTRY/shopverse-helmchart/shopverse \
                --namespace shopverse \
                --create-namespace \
                --set frontend.image=$FRONTEND_IMAGE \
                --set backend.image=$BACKEND_IMAGE \
                --wait
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                kubectl get nodes
                kubectl get pods -n shopverse
                kubectl get svc -n shopverse
                kubectl get ingress -n shopverse
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment Successful'
        }

        failure {
            echo 'Deployment Failed'
        }
    }
}
