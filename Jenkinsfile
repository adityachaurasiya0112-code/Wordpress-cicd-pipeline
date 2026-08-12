```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(false)
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        NAMESPACE = 'wp-project'
        KUBECTL_VERSION = 'v1.34.1'
        KUBECTL_BIN = "${WORKSPACE}/kubectl"
        KUBECONFIG = "${WORKSPACE}/kubeconfig"
    }

    stages {

        stage('Checkout') {
            steps {
                echo '==> Checking out source code'
                checkout scm
            }
        }

        stage('Install kubectl') {
            steps {
                sh '''
                    set -eu

                    echo "==> Installing kubectl locally in Jenkins workspace"

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

                    echo "==> Checking Kubernetes configuration"

                    if [ -n "${KUBECONFIG_FILE:-}" ] && [ -f "$KUBECONFIG_FILE" ]; then
                        cp "$KUBECONFIG_FILE" "$KUBECONFIG"
                        chmod 600 "$KUBECONFIG"
                    elif [ -f "/var/lib/jenkins/.kube/config" ]; then
                        cp "/var/lib/jenkins/.kube/config" "$KUBECONFIG"
                        chmod 600 "$KUBECONFIG"
                    elif [ -f "$HOME/.kube/config" ]; then
                        cp "$HOME/.kube/config" "$KUBECONFIG"
                        chmod 600 "$KUBECONFIG"
                    else
                        echo "ERROR: Kubernetes kubeconfig was not found."
                        echo ""
                        echo "Configure KUBECONFIG_FILE as a Jenkins secret/file"
                        echo "or make a kubeconfig available to the Jenkins user."
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

                    echo "==> Files found:"
                    ls -lh \
                        namespace-secret.yaml \
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

    print(file + " OK")
PY
                '''
            }
        }

        stage('Kubernetes Dry Run') {
            steps {
                sh '''
                    set -eu

                    echo "==> Running Kubernetes server-side dry run"

                    "$KUBECTL_BIN" apply \
                        --dry-run=server \
                        -f namespace-secret.yaml

                    "$KUBECTL_BIN" apply \
                        --dry-run=server \
                        -f mysql-deployment.yaml

                    "$KUBECTL_BIN" apply \
                        --dry-run=server \
                        -f wordpress-deployment.yaml

                    echo "==> Dry run successful"
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

                    "$KUBECTL_BIN" get namespace "$NAMESPACE"
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

                    echo "======================================"
                    echo "Pods"
                    echo "======================================"

                    "$KUBECTL_BIN" get pods \
                        -n "$NAMESPACE" \
                        -o wide

                    echo "======================================"
                    echo "Deployments"
                    echo "======================================"

                    "$KUBECTL_BIN" get deployments \
                        -n "$NAMESPACE"

                    echo "======================================"
                    echo "Services"
                    echo "======================================"

                    "$KUBECTL_BIN" get services \
                        -n "$NAMESPACE"

                    echo "======================================"
                    echo "WordPress Service"
                    echo "======================================"

                    "$KUBECTL_BIN" get svc wordpress-service \
                        -n "$NAMESPACE"
                '''
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo '✅ WORDPRESS CI/CD SUCCESSFUL'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo '❌ PIPELINE FAILED'
            echo '======================================'
            echo 'Check the failed stage above.'
        }

        always {
            cleanWs()
        }
    }
}
```
