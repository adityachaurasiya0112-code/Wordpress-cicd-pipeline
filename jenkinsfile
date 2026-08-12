pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        NAMESPACE = 'wp-project'
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
    }

    stages {

        stage('Checkout') {
            steps {
                echo '========== CHECKOUT =========='

                git(
                    branch: 'main',
                    url: 'https://github.com/ayu-haker/wordpress-ci-cd-pipeline.git'
                )
            }
        }

        stage('Check Kubernetes') {
            steps {
                echo '========== KUBERNETES CHECK =========='

                sh '''
                    set -e

                    echo "kubectl version:"
                    kubectl version --client

                    echo ""
                    echo "Kubernetes nodes:"
                    kubectl get nodes

                    echo ""
                    echo "Current context:"
                    kubectl config current-context
                '''
            }
        }

        stage('Validate Manifests') {
            steps {
                echo '========== VALIDATE MANIFESTS =========='

                sh '''
                    set -e

                    echo "Validating namespace and secret..."
                    kubectl apply \
                        --dry-run=client \
                        -f namespace-secret.yaml

                    echo "Validating MySQL..."
                    kubectl apply \
                        --dry-run=client \
                        -f mysql-deployment.yaml

                    echo "Validating WordPress..."
                    kubectl apply \
                        --dry-run=client \
                        -f wordpress-deployment.yaml

                    echo "✅ All manifests are valid"
                '''
            }
        }

        stage('Deploy Namespace & Secret') {
            steps {
                echo '========== DEPLOY NAMESPACE & SECRET =========='

                sh '''
                    set -e

                    kubectl apply -f namespace-secret.yaml

                    echo "Namespace:"
                    kubectl get namespace ${NAMESPACE}

                    echo "Secret:"
                    kubectl -n ${NAMESPACE} get secret
                '''
            }
        }

        stage('Deploy MySQL') {
            steps {
                echo '========== DEPLOY MYSQL =========='

                sh '''
                    set -e

                    kubectl apply -f mysql-deployment.yaml

                    echo "Waiting for MySQL..."

                    kubectl -n ${NAMESPACE} rollout status \
                        deployment/mysql \
                        --timeout=300s

                    echo "✅ MySQL deployment successful"

                    kubectl -n ${NAMESPACE} get pods
                '''
            }
        }

        stage('Deploy WordPress') {
            steps {
                echo '========== DEPLOY WORDPRESS =========='

                sh '''
                    set -e

                    kubectl apply -f wordpress-deployment.yaml

                    echo "Waiting for WordPress..."

                    kubectl -n ${NAMESPACE} rollout status \
                        deployment/wordpress \
                        --timeout=300s

                    echo "✅ WordPress deployment successful"
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                echo '========== VERIFY DEPLOYMENT =========='

                sh '''
                    set -e

                    echo ""
                    echo "===== NODES ====="
                    kubectl get nodes

                    echo ""
                    echo "===== PODS ====="
                    kubectl -n ${NAMESPACE} get pods -o wide

                    echo ""
                    echo "===== SERVICES ====="
                    kubectl -n ${NAMESPACE} get svc

                    echo ""
                    echo "===== DEPLOYMENTS ====="
                    kubectl -n ${NAMESPACE} get deployments

                    echo ""
                    echo "===== PVC ====="
                    kubectl -n ${NAMESPACE} get pvc

                    echo ""
                    echo "===== MYSQL STATUS ====="
                    kubectl -n ${NAMESPACE} rollout status \
                        deployment/mysql \
                        --timeout=60s

                    echo ""
                    echo "===== WORDPRESS STATUS ====="
                    kubectl -n ${NAMESPACE} rollout status \
                        deployment/wordpress \
                        --timeout=60s

                    echo ""
                    echo "======================================"
                    echo "✅ WORDPRESS DEPLOYMENT VERIFIED"
                    echo "======================================"
                '''
            }
        }
    }

    post {
        success {
            echo '''
=========================================
✅ WORDPRESS CI/CD PIPELINE SUCCESS
=========================================
Namespace: wp-project
Kubernetes: Minikube
Deployment: Successful
=========================================
'''
        }

        failure {
            echo '''
=========================================
❌ WORDPRESS CI/CD PIPELINE FAILED
=========================================
Check the stage logs above.
=========================================
'''
        }

        always {
            cleanWs()
        }
    }
}
