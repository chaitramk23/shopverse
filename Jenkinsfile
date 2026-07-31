pipeline {
    agent any

    environment {
        AWS_REGION       = 'us-east-1'
        ECR_REGISTRY     = '835505307872.dkr.ecr.us-east-1.amazonaws.com'
        EKS_CLUSTER_NAME = 'chaitra-cluster1'

        FRONTEND_IMAGE = "${ECR_REGISTRY}/shopverse-frontend:${BUILD_NUMBER}"
        BACKEND_IMAGE  = "${ECR_REGISTRY}/shopverse-backend:${BUILD_NUMBER}"
    }

    stages {

        stage('Backend Tests') {
            steps {
                dir('backend') {
                    sh '''
                    go mod tidy
                    export CGO_ENABLED=0
                    go test ./...
                    '''
                }
            }
        }

        stage('Frontend Lint') {
            steps {
                dir('frontend') {
                    sh '''
                    npm install
                    npx eslint . --ext js,jsx || true
                    '''
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
                trivy image --exit-code 1 --severity CRITICAL shopverse-frontend:scan

                trivy image --exit-code 1 --severity CRITICAL shopverse-backend:scan
                '''
            }
        }

        stage('ECR Login') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                    aws sts get-caller-identity

                    aws ecr get-login-password --region us-east-1 | \
                    docker login --username AWS \
                    --password-stdin 835505307872.dkr.ecr.us-east-1.amazonaws.com
                    '''
                }
            }
        }

        stage('ECR Login') {
            steps {
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
                -t $ECR_REGISTRY/shopverse-frontend:latest \
                ./frontend

                docker build -t $BACKEND_IMAGE \
                -t $ECR_REGISTRY/shopverse-backend:latest \
                ./backend
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

        stage('Update Kubeconfig') {
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    sh '''
                    aws eks update-kubeconfig \
                    --name $EKS_CLUSTER_NAME \
                    --region $AWS_REGION
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                kubectl apply -f k8s/namespace.yaml || true

                kubectl apply -f k8s/mysql.yaml

                kubectl apply -f k8s/backend-deployment.yaml
                kubectl apply -f k8s/backend-service.yaml

                kubectl apply -f k8s/frontend-deployment.yaml
                kubectl apply -f k8s/frontend-service.yaml

                kubectl apply -f k8s/ingress.yaml
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
            echo 'ShopVerse Deployment Successful'
        }

        failure {
            echo 'ShopVerse Deployment Failed'
        }
    }
}
