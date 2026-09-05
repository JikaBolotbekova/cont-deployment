pipeline {
    agent any

    environment {
        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:${env.PATH}"

        AWS_REGION = "us-east-1"
        EKS_CLUSTER = "eks-test"

        NAMESPACE = "restaurant-prod"
        DEPLOYMENT_NAME = "restaurant-company"
    }

    stages {

        stage('1. Checkout') {
            steps {
                echo 'Checking out CD repository...'
                checkout scm
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

        stage('3. AWS Authentication') {
            steps {
                withCredentials([
                    aws(
                        credentialsId: 'aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        echo "========== AWS AUTHENTICATION =========="
                        aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('4. Connect to EKS') {
            steps {
                withCredentials([
                    aws(
                        credentialsId: 'aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        echo "========== CONNECT TO EKS =========="

                        aws eks update-kubeconfig \
                            --region $AWS_REGION \
                            --name $EKS_CLUSTER

                        echo "Current Kubernetes context:"
                        kubectl config current-context

                        echo "Cluster nodes:"
                        kubectl get nodes
                    '''
                }
            }
        }

        stage('5. Create Namespace') {
            steps {
                sh '''
                    echo "========== CREATE NAMESPACE =========="
                    kubectl apply -f namespace.yaml
                '''
            }
        }

        stage('6. Apply Configuration') {
            steps {
                sh '''
                    echo "========== APPLY CONFIGURATION =========="

                    kubectl apply -f configmap.yaml -n $NAMESPACE
                    kubectl apply -f secrets.yaml -n $NAMESPACE
                    kubectl apply -f serviceaccount.yaml -n $NAMESPACE
                '''
            }
        }

        stage('7. Deploy Application') {
            steps {
                sh '''
                    echo "========== DEPLOY APPLICATION =========="

                    kubectl apply -f deployment.yaml -n $NAMESPACE
                    kubectl apply -f service.yaml -n $NAMESPACE
                '''
            }
        }

        stage('8. Apply Production Resources') {
            steps {
                sh '''
                    echo "========== PRODUCTION RESOURCES =========="

                    if [ -f hpa.yaml ]; then
                        kubectl apply -f hpa.yaml -n $NAMESPACE
                    else
                        echo "hpa.yaml not found - skipping"
                    fi

                    if [ -f pdb.yaml ]; then
                        kubectl apply -f pdb.yaml -n $NAMESPACE
                    else
                        echo "pdb.yaml not found - skipping"
                    fi

                    if [ -f networkpolicy.yaml ]; then
                        kubectl apply -f networkpolicy.yaml -n $NAMESPACE
                    else
                        echo "networkpolicy.yaml not found - skipping"
                    fi
                '''
            }
        }

        stage('9. Apply Ingress') {
            steps {
                sh '''
                    echo "========== APPLY INGRESS =========="

                    if [ -f ingress.yaml ]; then
                        kubectl apply -f ingress.yaml -n $NAMESPACE
                    else
                        echo "ingress.yaml not found - skipping"
                    fi
                '''
            }
        }

        stage('10. Verify Rollout') {
            steps {
                sh '''
                    echo "========== VERIFY ROLLOUT =========="

                    kubectl rollout status \
                        deployment/$DEPLOYMENT_NAME \
                        -n $NAMESPACE \
                        --timeout=300s
                '''
            }
        }

        stage('11. Verify Deployment') {
            steps {
                sh '''
                    echo "========== PODS =========="
                    kubectl get pods -n $NAMESPACE -o wide

                    echo "========== DEPLOYMENTS =========="
                    kubectl get deployments -n $NAMESPACE

                    echo "========== SERVICES =========="
                    kubectl get services -n $NAMESPACE

                    echo "========== INGRESS =========="
                    kubectl get ingress -n $NAMESPACE || true

                    echo "========== HPA =========="
                    kubectl get hpa -n $NAMESPACE || true
                '''
            }
        }
    }

    post {

        success {
            echo '======================================'
            echo 'DEPLOYMENT SUCCESSFUL'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'DEPLOYMENT FAILED'
            echo '======================================'

            sh '''
                echo "========== TROUBLESHOOTING =========="

                kubectl get pods -n $NAMESPACE || true

                echo "========== RECENT EVENTS =========="
                kubectl get events \
                    -n $NAMESPACE \
                    --sort-by=.metadata.creationTimestamp \
                    | tail -30 || true
            '''
        }

        always {
            echo 'Pipeline finished.'
        }
    }
}
