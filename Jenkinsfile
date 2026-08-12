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
                checkout scm
            }
        }

        stage('Check Tools') {
            steps {
                sh '''
                    set -e

                    echo "==> Checking Git"
                    git --version

                    echo "==> Checking kubectl"
                    kubectl version --client

                    echo "==> Checking Kubernetes connection"
                    kubectl cluster-info
                '''
            }
        }

        stage('Validate Manifests') {
            steps {
                sh '''
                    set -e

                    echo "==> Checking manifest files"

                    test -f namespace-secret.yaml
                    test -f mysql-deployment.yaml
                    test -f wordpress-deployment.yaml

                    ls -l namespace-secret.yaml \
                          mysql-deployment.yaml \
                          wordpress-deployment.yaml

                    echo "==> Validating YAML"

                    python3 - <<'PY'
import yaml

files = [
    "namespace-secret.yaml",
    "mysql-deployment.yaml",
    "wordpress-deployment.yaml"
]

for file in files:
    with open(file, "r") as f:
        list(yaml.safe_load_all(f))
    print(f"{file} OK")
PY
                '''
            }
        }

        stage('Kubernetes Dry Run') {
            steps {
                sh '''
                    set -e

                    echo "==> Kubernetes server-side dry run"

                    kubectl apply \
                        --dry-run=server \
                        -f namespace-secret.yaml

                    kubectl apply \
                        --dry-run=server \
                        -f mysql-deployment.yaml

                    kubectl apply \
                        --dry-run=server \
                        -f wordpress-deployment.yaml

                    echo "==> Dry run successful"
                '''
            }
        }

        stage('Deploy Namespace and Secret') {
            steps {
                sh '''
                    set -e

                    echo "==> Creating namespace and secret"

                    kubectl apply -f namespace-secret.yaml

                    echo "==> Namespace status"
                    kubectl get namespace ${NAMESPACE}
                '''
            }
        }

        stage('Deploy MySQL') {
            steps {
                sh '''
                    set -e

                    echo "==> Deploying MySQL"

                    kubectl apply -f mysql-deployment.yaml

                    echo "==> Waiting for MySQL"

                    kubectl rollout status \
                        deployment/mysql \
                        -n ${NAMESPACE} \
                        --timeout=180s
                '''
            }
        }

        stage('Deploy WordPress') {
            steps {
                sh '''
                    set -e

                    echo "==> Deploying WordPress"

                    kubectl apply -f wordpress-deployment.yaml

                    echo "==> Waiting for WordPress"

                    kubectl rollout status \
                        deployment/wordpress \
                        -n ${NAMESPACE} \
                        --timeout=180s
                '''
            }
        }

        stage('Post Deploy Verification') {
            steps {
                sh '''
                    set -e

                    echo "================================"
                    echo "Pods"
                    echo "================================"

                    kubectl get pods \
                        -n ${NAMESPACE} \
                        -o wide

                    echo "================================"
                    echo "Deployments"
                    echo "================================"

                    kubectl get deployments \
                        -n ${NAMESPACE}

                    echo "================================"
                    echo "Services"
                    echo "================================"

                    kubectl get services \
                        -n ${NAMESPACE}

                    echo "================================"
                    echo "WordPress Endpoints"
                    echo "================================"

                    kubectl get svc wordpress-service \
                        -n ${NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo '✅ WordPress CI/CD pipeline completed successfully!'
        }

        failure {
            echo '❌ Pipeline failed. Check the stage logs above.'
        }

        always {
            cleanWs()
        }
    }
}
