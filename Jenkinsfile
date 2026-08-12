pipeline {
    agent any

    environment {
        NAMESPACE = 'wp-project'
        // agent pe kubectl na ho to 'Ensure kubectl' stage isse workspace/bin mein download kar deta hai
        PATH = "${WORKSPACE}/bin:${PATH}"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/adityachaurasiya0112-code/Wordpress-cicd-pipeline.git'
            }
        }

        stage('Ensure kubectl') {
            steps {
                sh '''
                    if ! command -v kubectl >/dev/null 2>&1; then
                        echo "kubectl not found, installing locally into workspace..."
                        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                        mkdir -p bin
                        mv kubectl bin/
                    else
                        echo "kubectl already available: $(command -v kubectl)"
                    fi
                '''
            }
        }

        stage('Validate Manifests') {
            steps {
                sh '''
                    echo "Validating Kubernetes YAML files..."
                    kubectl apply --dry-run=client -f namespace-secret.yaml
                    kubectl apply --dry-run=client -f mysql-deployment.yaml
                    kubectl apply --dry-run=client -f wordpress-deployment.yaml
                '''
            }
        }

        stage('Deploy Namespace & Secret') {
            steps {
                sh 'kubectl apply -f namespace-secret.yaml'
            }
        }

        stage('Deploy MySQL') {
            steps {
                sh 'kubectl apply -f mysql-deployment.yaml'
            }
        }

        stage('Deploy WordPress') {
            steps {
                sh 'kubectl apply -f wordpress-deployment.yaml'
            }
        }

        stage('Verify Rollout') {
            steps {
                sh '''
                    kubectl rollout status deployment/mysql -n ${NAMESPACE} --timeout=120s || true
                    kubectl rollout status deployment/wordpress -n ${NAMESPACE} --timeout=120s
                '''
            }
        }

        stage('Show Service Info') {
            steps {
                sh '''
                    echo "Deployed resources:"
                    kubectl get pods -n ${NAMESPACE}
                    kubectl get svc -n ${NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo '✅ WordPress CI/CD pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Check logs above for the failing stage.'
        }
        always {
            cleanWs()
        }
    }
}
