pipeline {
    agent any

    environment {
        // macOS Homebrew tools
        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:${env.PATH}"

        // AWS / EKS
        AWS_REGION = "us-east-1"
        EKS_CLUSTER = "eks-test"

        // Kubernetes
        NAMESPACE = "restaurant-prod"
        DEPLOYMENT_NAME = "restaurant-company"
    }

    stages {

        stage('1. Checkout') {
            steps {
                echo 'Checking out CD repository...'
                checkout scm

                sh '''
                    echo "========== REPOSITORY FILES =========="
                    pwd
                    ls -la
                '''
            }
        }

        stage('2. Verify Tools') {
            steps {
                sh '''
                    echo "========== VERIFY TOOLS =========="

                    echo "AWS CLI:"
                    which aws
                    aws --version

                    echo "kubectl:"
                    which kubectl
                    kubectl version --client

                    echo "Git:"
                    which git
                    git --version
                '''
            }
        }

        stage('3. Deploy to AWS EKS') {
            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {

                    sh '''
                        set -e

                        echo "========================================="
                        echo "AWS AUTHENTICATION"
                        echo "========================================="

                        aws sts get-caller-identity

                        echo ""
                        echo "========================================="
                        echo "CHECK EKS CLUSTER"
                        echo "========================================="

                        aws eks describe-cluster \
                            --region "$AWS_REGION" \
                            --name "$EKS_CLUSTER" \
                            --query 'cluster.{Name:name,Status:status,Version:version}' \
                            --output table

                        echo ""
                        echo "========================================="
                        echo "CONNECT TO EKS"
                        echo "========================================="

                        aws eks update-kubeconfig \
                            --region "$AWS_REGION" \
                            --name "$EKS_CLUSTER"

                        echo ""
                        echo "Current Kubernetes context:"
                        kubectl config current-context

                        echo ""
                        echo "Worker nodes:"
                        kubectl get nodes -o wide

                        echo ""
                        echo "========================================="
                        echo "CREATE NAMESPACE"
                        echo "========================================="

                        kubectl apply -f namespace.yaml

                        echo ""
                        echo "========================================="
                        echo "APPLY CONFIGURATION"
                        echo "========================================="

                        kubectl apply -f configmap.yaml \
                            -n "$NAMESPACE"

                        kubectl apply -f secrets.yaml \
                            -n "$NAMESPACE"

                        kubectl apply -f serviceaccount.yaml \
                            -n "$NAMESPACE"

                        echo ""
                        echo "========================================="
                        echo "DEPLOY APPLICATION"
                        echo "========================================="

                        kubectl apply -f deployment.yaml \
                            -n "$NAMESPACE"

                        kubectl apply -f service.yaml \
                            -n "$NAMESPACE"

                        echo ""
                        echo "========================================="
                        echo "APPLY PRODUCTION RESOURCES"
                        echo "========================================="

                        kubectl apply -f hpa.yaml \
                            -n "$NAMESPACE"

                        kubectl apply -f pdb.yaml \
                            -n "$NAMESPACE"

                        kubectl apply -f networkpolicy.yaml \
                            -n "$NAMESPACE"

                        echo ""
                        echo "========================================="
                        echo "APPLY INGRESS"
                        echo "========================================="

                        kubectl apply -f ingress.yaml \
                            -n "$NAMESPACE"

                        echo ""
                        echo "========================================="
                        echo "WAIT FOR DEPLOYMENT"
                        echo "========================================="

                        kubectl rollout status \
                            deployment/"$DEPLOYMENT_NAME" \
                            -n "$NAMESPACE" \
                            --timeout=300s

                        echo ""
                        echo "========================================="
                        echo "VERIFY DEPLOYMENT"
                        echo "========================================="

                        echo ""
                        echo "PODS:"
                        kubectl get pods \
                            -n "$NAMESPACE" \
                            -o wide

                        echo ""
                        echo "DEPLOYMENTS:"
                        kubectl get deployments \
                            -n "$NAMESPACE"

                        echo ""
                        echo "SERVICES:"
                        kubectl get services \
                            -n "$NAMESPACE"

                        echo ""
                        echo "INGRESS:"
                        kubectl get ingress \
                            -n "$NAMESPACE"

                        echo ""
                        echo "HPA:"
                        kubectl get hpa \
                            -n "$NAMESPACE"

                        echo ""
                        echo "========================================="
                        echo "DEPLOYMENT COMPLETED"
                        echo "========================================="
                    '''
                }
            }
        }
    }

    post {

        success {
            echo '========================================='
            echo 'DEPLOYMENT SUCCESSFUL'
            echo 'Restaurant Company deployed to AWS EKS.'
            echo '========================================='
        }

        failure {
            echo '========================================='
            echo 'DEPLOYMENT FAILED'
            echo 'Check the first error in the Jenkins console.'
            echo '========================================='
        }

        always {
            echo 'Pipeline finished.'
        }
    }
}
