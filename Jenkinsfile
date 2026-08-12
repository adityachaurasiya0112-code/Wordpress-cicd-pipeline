pipeline {
    agent any

    environment {
        // Kubeconfig credential ID configured in Jenkins (Manage Jenkins > Credentials)
        KUBECONFIG_CRED_ID = 'kubeconfig-cred'
        NAMESPACE           = 'wp-project'
        GIT_REPO            = 'https://github.com/adityachaurasiya0112-code/Wordpress-cicd-pipeline.git'
        GIT_BRANCH           = 'main'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO}"
            }
        }

        stage('Validate Manifests') {
            steps {
                sh '''
                    echo "Validating YAML syntax..."
                    kubectl apply --dry-run=client -f namespace-secret.yaml
                    kubectl apply --dry-run=client -f mysql-deployment.yaml
                    kubectl apply --dry-run=client -f wordpress-deployment.yaml
                '''
            }
        }

        stage('Deploy Namespace & Secrets') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED_ID}", variable: 'KUBECONFIG')]) {
                    sh 'kubectl apply -f namespace-secret.yaml'
                }
            }
        }

        stage('Deploy MySQL') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED_ID}", variable: 'KUBECONFIG')]) {
                    sh 'kubectl apply -f mysql-deployment.yaml'
                    sh "kubectl rollout status deployment/mysql -n ${NAMESPACE} --timeout=120s"
                }
            }
        }

        stage('Deploy WordPress') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED_ID}", variable: 'KUBECONFIG')]) {
                    sh 'kubectl apply -f wordpress-deployment.yaml'
                    sh "kubectl rollout status deployment/wordpress -n ${NAMESPACE} --timeout=180s"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED_ID}", variable: 'KUBECONFIG')]) {
                    sh """
                        echo "Pods:"
                        kubectl get pods -n ${NAMESPACE} -o wide
                        echo "Services:"
                        kubectl get svc -n ${NAMESPACE}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ WordPress CI/CD pipeline completed successfully."
        }
        failure {
            echo "❌ Pipeline failed. Check logs above for the failing stage."
        }
        always {
            cleanWs()
        }
    }
}
