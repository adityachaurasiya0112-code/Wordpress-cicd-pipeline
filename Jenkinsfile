pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        NAMESPACE = 'wp-project'
        KUBECTL_VERSION = 'v1.34.1'
        KUBECTL_BIN = "${WORKSPACE}/kubectl"
        KUBECONFIG = "${WORKSPACE}/kubeconfig"
    }

    stages {

        stage('Install kubectl') {
            steps {
                sh '''
                    set -eu

                    echo "==> Installing kubectl"

                    if [ ! -x "$KUBECTL_BIN" ]; then
                        curl -fL \
                          "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" \
                          -o "$KUBECTL_BIN"

                        chmod +x "$KUBECTL_BIN"
                    fi

                    "$KUBECTL_BIN" version --client
                '''
            }
        }

        stage('Check Kubernetes Access') {
            steps {
                sh '''
                    set -eu

                    echo "==> Checking kubeconfig"

                    if [ -f "/var/lib/jenkins/.kube/config" ]; then
                        cp "/var/lib/jenkins/.kube/config" "$KUBECONFIG"
                        chmod 600 "$KUBECONFIG"

                    elif [ -f "$HOME/.kube/config" ]; then
                        cp "$HOME/.kube/config" "$KUBECONFIG"
                        chmod 600 "$KUBECONFIG"

                    else
                        echo "ERROR: kubeconfig not found"
                        exit 1
                    fi

                    echo "==> Testing Kubernetes connection"

                    "$KUBECTL_BIN" cluster-info
                    "$KUBECTL_BIN" get nodes
                '''
            }
        }

        stage('Validate Manifests') {
            steps {
                sh '''
                    set -eu

                    echo "==> Checking manifest files"

                    test -f namespace-secret.yaml
                    test -f mysql-deployment.yaml
                    test -f wordpress-deployment.yaml

                    echo "==> Manifest files found"

                    ls -lh \
                        namespace-secret.yaml \
                        mysql-deployment.yaml \
                        wordpress-deployment.yaml

                    echo "==> Validating YAML syntax"

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

    print(file + " OK")
PY
                '''
            }
        }

        stage('Kubernetes Dry Run') {
            steps {
                sh '''
                    set -eu

                    echo "==> Validating Namespace"

                    "$KUBECTL_BIN" apply \
                        --dry-run=client \
                        -f namespace-secret.yaml

                    echo "==> Validating MySQL"

                    "$KUBECTL_BIN" apply \
                        --dry-run=client \
                        -f mysql-deployment.yaml

                    echo "==> Validating WordPress"

                    "$KUBECTL_BIN" apply \
                        --dry-run=client \
                        -f wordpress-deployment.yaml

                    echo "==> Client-side dry run successful"
                '''
            }
        }

        stage('Deploy Namespace and Secret') {
            steps {
                sh '''
                    set -eu

                    echo "==> Deploying namespace and secret"

                    "$KUBECTL_BIN" apply \
                        -f namespace-secret.yaml

                    echo "==> Namespace"

                    "$KUBECTL_BIN" get namespace "$NAMESPACE"

                    echo "==> Secrets"

                    "$KUBECTL_BIN" get secrets \
                        -n "$NAMESPACE"
                '''
            }
        }

        stage('Deploy MySQL') {
            steps {
                sh '''
                    set -eu

                    echo "==> Deploying MySQL"

                    "$KUBECTL_BIN" apply \
                        -f mysql-deployment.yaml

                    echo "==> Waiting for MySQL rollout"

                    "$KUBECTL_BIN" rollout status \
                        deployment/mysql \
                        -n "$NAMESPACE" \
                        --timeout=180s
                '''
            }
        }

        stage('Deploy WordPress') {
            steps {
                sh '''
                    set -eu

                    echo "==> Deploying WordPress"

                    "$KUBECTL_BIN" apply \
                        -f wordpress-deployment.yaml

                    echo "==> Waiting for WordPress rollout"

                    "$KUBECTL_BIN" rollout status \
                        deployment/wordpress \
                        -n "$NAMESPACE" \
                        --timeout=180s
                '''
            }
        }

        stage('Post Deploy Verification') {
            steps {
                sh '''
                    set -eu

                    echo "========== PODS =========="

                    "$KUBECTL_BIN" get pods \
                        -n "$NAMESPACE" \
                        -o wide

                    echo "========== DEPLOYMENTS =========="

                    "$KUBECTL_BIN" get deployments \
                        -n "$NAMESPACE"

                    echo "========== SERVICES =========="

                    "$KUBECTL_BIN" get services \
                        -n "$NAMESPACE"

                    echo "========== PVC =========="

                    "$KUBECTL_BIN" get pvc \
                        -n "$NAMESPACE" || true
                '''
            }
        }
    }

    post {
        success {
            echo '=========================================='
            echo '✅ WORDPRESS CI/CD PIPELINE SUCCESSFUL'
            echo '=========================================='
        }

        failure {
            echo '=========================================='
            echo '❌ WORDPRESS CI/CD PIPELINE FAILED'
            echo '=========================================='
        }

        always {
            cleanWs()
        }
    }
}
