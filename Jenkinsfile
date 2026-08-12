pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        NAMESPACE   = 'wp-project'
        KUBECONFIG_CRED = 'kubeconfig-cred'   // Jenkins credential ID (Secret file, type: kubeconfig)
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Manifests') {
            steps {
                sh '''
                    echo "==> Checking manifest files exist"
                    ls -l namespace-secret.yaml mysql-deployment.yaml wordpress-deployment.yaml

                    echo "==> YAML syntax check"
                    for f in namespace-secret.yaml mysql-deployment.yaml wordpress-deployment.yaml; do
                        python3 -c "import yaml,sys; list(yaml.safe_load_all(open('$f')))" && echo "$f OK"
                    done
                '''
            }
        }

        stage('Dry-Run Apply (Server-side check)') {
            steps {
                withKubeConfig(credentialsId: "${KUBECONFIG_CRED}") {
                    sh '''
                        kubectl apply --dry-run=server -f namespace-secret.yaml
                        kubectl apply --dry-run=server -f mysql-deployment.yaml
                        kubectl apply --dry-run=server -f wordpress-deployment.yaml
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(credentialsId: "${KUBECONFIG_CRED}") {
                    sh '''
                        echo "==> Creating namespace + secret first"
                        kubectl apply -f namespace-secret.yaml

                        echo "==> Deploying MySQL"
                        kubectl apply -f mysql-deployment.yaml

                        echo "==> Waiting for MySQL rollout"
                        kubectl rollout status deployment/mysql -n ${NAMESPACE} --timeout=120s

                        echo "==> Deploying WordPress"
                        kubectl apply -f wordpress-deployment.yaml

                        echo "==> Waiting for WordPress rollout"
                        kubectl rollout status deployment/wordpress -n ${NAMESPACE} --timeout=180s
                    '''
                }
            }
        }

        stage('Post-Deploy Verification') {
            steps {
                withKubeConfig(credentialsId: "${KUBECONFIG_CRED}") {
                    sh '''
                        echo "==> Pods:"
                        kubectl get pods -n ${NAMESPACE} -o wide

                        echo "==> Services:"
                        kubectl get svc -n ${NAMESPACE}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ WordPress CI/CD pipeline completed successfully."
        }
        failure {
            echo "❌ Pipeline failed. Rolling back is manual — check kubectl rollout undo if needed."
        }
        always {
            cleanWs()
        }
    }
}
