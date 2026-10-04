pipeline {
    agent any

    environment {
        AWS_REGION    = 'ap-south-1'
        AWS_ACCOUNT_ID = '986918902913'
        ECR_REPO      = 'boutique-frontend'
        IMAGE_URI     = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        IMAGE_TAG     = "${BUILD_NUMBER}"
        EKS_CLUSTER   = 'devops-project1'
        NAMESPACE     = 'boutique'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Source') {
            steps {
                sh '''
                    set -e
                    helm lint ./helm-chart
                    test -f helm-aws-values.yaml
                    test -d src/frontend
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -e
                    docker build -t ${IMAGE_URI}:${IMAGE_TAG} ./src/frontend
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    trivy image --severity HIGH,CRITICAL --exit-code 0 ${IMAGE_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('Push ECR') {
            steps {
                sh '''
                    set -e
                    aws ecr get-login-password --region ${AWS_REGION} | \
                      docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    docker push ${IMAGE_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy with Helm') {
            steps {
                sh '''
                    set -e

                    aws eks update-kubeconfig \
                      --region ${AWS_REGION} \
                      --name ${EKS_CLUSTER}

                    helm upgrade --install boutique ./helm-chart \
                      --namespace ${NAMESPACE} \
                      --create-namespace \
                      -f helm-aws-values.yaml \
                      --set images.repository=us-central1-docker.pkg.dev/online-boutique-ci/microservices-demo \
                      --set images.tag="" \
                      --set frontend.imageRepository=${IMAGE_URI} \
                      --set frontend.imageTag=${IMAGE_TAG}
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    set -e

                    kubectl rollout status deployment/frontend \
                      -n ${NAMESPACE} \
                      --timeout=300s

                    kubectl get pods -n ${NAMESPACE}

                    kubectl get svc frontend-external \
                      -n ${NAMESPACE}
                '''
            }
        }
    }
}
