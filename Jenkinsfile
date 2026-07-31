pipeline {
agent any

```
environment {
    AWS_REGION       = 'us-east-1'
    EKS_CLUSTER_NAME = 'chaitra-cluster1'
    ECR_REGISTRY     = '835505307872.dkr.ecr.us-east-1.amazonaws.com'

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
            trivy image --exit-code 0 --severity HIGH,CRITICAL shopverse-frontend:scan
            trivy image --exit-code 0 --severity HIGH,CRITICAL shopverse-backend:scan
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

                aws ecr get-login-password --region $AWS_REGION | \
                docker login --username AWS \
                --password-stdin $ECR_REGISTRY
                '''
            }
        }
    }

    stage('Build & Tag Images') {
        steps {
            sh '''
            docker build \
            -t $FRONTEND_IMAGE \
            -t $ECR_REGISTRY/shopverse-frontend:latest \
            ./frontend

            docker build \
            -t $BACKEND_IMAGE \
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
            withCredentials([[
                $class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: 'aws-creds'
            ]]) {
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
            helm upgrade --install shopverse ./helm/shopverse \
            --namespace shopverse \
            --create-namespace \
            --wait
            '''
        }
    }

    stage('Verify Deployment') {
        steps {
            sh '''
            echo "===== Nodes ====="
            kubectl get nodes

            echo "===== Pods ====="
            kubectl get pods -n shopverse

            echo "===== Services ====="
            kubectl get svc -n shopverse

            echo "===== Ingress ====="
            kubectl get ingress -n shopverse

            echo "===== Helm Release ====="
            helm list -n shopverse
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
```

}
