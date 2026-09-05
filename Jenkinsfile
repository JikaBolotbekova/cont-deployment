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

                sh '''
                    echo "========== FILES IN WORKSPACE =========="
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

        stage('3. AWS Authentication') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
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
                    usernamePassword(
                        credentialsId: 'aws-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        echo "========== CONNECT TO EKS =========="

                        aws eks update-kubeconfig \
                            --region $AWS_REGION \
                            --name $EKS_CLUSTER

                        echo "Current Kubernetes context:"
                        kubectl config current-context

                        echo "Worker nodes:"
                        kubectl get nodes
                    '''
                }
            }
        }

        stage('5. Create Namespace') {
            steps {
                sh '''
                    echo "========== NAMESPACE =========="
                    kubectl apply -f namespace.yaml
                    kubectl get namespace $NAMESPACE
                '''
            }
        }

        stage('6. Apply Configuration') {
            steps {
                sh '''
                    echo "========== CONFIGURATION =========="

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

                    kubectl apply -f pdb.yaml -n $NAMESPACE
                    kubectl apply -f networkpolicy.yaml -n $NAMESPACE
                '''
            }
        }

        stage('9. Apply Ingress') {
            steps {
                sh '''
                    echo "========== INGRESS =========="

                    kubectl apply -f ingress.yaml -n $NAMESPACE
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

                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment -n $NAMESPACE

                    echo "========== SERVICE =========="
                    kubectl get service -n $NAMESPACE

                    echo "========== INGRESS =========="
                    kubectl get ingress -n $NAMESPACE

                    echo "========== PDB =========="
                    kubectl get pdb -n $NAMESPACE

                    echo "========== NETWORK POLICY =========="
                    kubectl get networkpolicy -n $NAMESPACE
                '''
            }
        }
    }

    post {

        success {
            echo '''
========================================
       DEPLOYMENT SUCCESSFUL
========================================
            '''
        }

        failure {
            echo '''
========================================
        DEPLOYMENT FAILED
========================================
Check Jenkins stage and Kubernetes events.
            '''

            sh '''
                echo "========== POD STATUS =========="
                kubectl get pods -n $NAMESPACE || true

                echo "========== RECENT EVENTS =========="
                kubectl get events \
                    -n $NAMESPACE \
                    --sort-by=.lastTimestamp | tail -30 || true
            '''
        }

        always {
            echo 'Pipeline finished.'
        }
    }
}
