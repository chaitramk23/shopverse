pipeline {
    agent any

    environment {
        TF_STATE_BUCKET  = 'shopverse-tf-statefile'
        ECR_REGISTRY     = '835505307872.dkr.ecr.us-east-1.amazonaws.com'
        EKS_CLUSTER_NAME = 'chaitra-cluster1'
        AWS_REGION       = 'us-east-1'

        FRONTEND_IMAGE = "${ECR_REGISTRY}/shopverse-frontend:${BUILD_NUMBER}"
        BACKEND_IMAGE  = "${ECR_REGISTRY}/shopverse-backend:${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/chaitramk23/shopverse.git'
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
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                sh '''
                aws ecr get-login-password --region us-east-1 | \
                docker login --username AWS \
                --password-stdin 835505307872.dkr.ecr.us-east-1.amazonaws.com
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
