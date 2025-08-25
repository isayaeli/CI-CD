pipeline {
     agent any
    
    environment {
        APP_NAME = "django-app"
        DOCKER_IMAGE = "${APP_NAME}:${env.BUILD_NUMBER}"
        K8S_NAMESPACE = "django-app"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                url: 'https://github.com/isayaeli/CI-CD.git'
            }
        }

        stage('Setup Minikube Environment') {
            steps {
                script {
                    // Install required tools
                    sh '''
                    # Update package list
                    apt update -y
                    
                    # Install Docker
                    apt install -y docker.io
                    
                    # Install kubectl
                    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
                    
                    # Install minikube if not already installed
                    if ! command -v minikube &> /dev/null; then
                        curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
                        install minikube-linux-amd64 /usr/local/bin/minikube
                    fi
                    
                    # Point Docker to Minikube's Docker daemon
                    eval $(minikube docker-env)
                    
                    # Verify installations
                    docker --version
                    kubectl version --client
                    minikube version
                    '''
                }
            }
        }


        // stage('Install Python') {
        //     steps {
        //         script {
        //             sh '''
        //             # Install Python3 if not available
        //             if ! command -v python3 &> /dev/null; then
        //                 echo "Installing Python3..."
        //                 apt update && apt install -y python3 python3-pip python3-venv
        //             fi
                    
        //             # Create symlink if python doesn't exist
        //             if ! command -v python &> /dev/null; then
        //                 ln -s $(which python3) /usr/bin/python
        //             fi
                    
        //             python --version
        //             '''
        //         }
        //     }
        // }

 
        
        stage('Install Dependencies') {
            steps {
                script {
                    sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt
                    '''
                }
            }
        }
        
        stage('Run Tests') {
            steps {
                script {
                    sh '''
                    source venv/bin/activate
                    python manage.py test --noinput
                    '''
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    // Build using Minikube's Docker daemon
                    sh "docker build -t ${DOCKER_IMAGE} ."
                    
                    // Also tag as latest for easy reference
                    sh "docker tag ${DOCKER_IMAGE} ${APP_NAME}:latest"
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Create namespace if it doesn't exist
                    sh """
                    kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                    """
                    
                    // Deploy or update the application
                    sh """
                    # Apply all Kubernetes manifests
                    kubectl apply -f k8s/ -n ${K8S_NAMESPACE}
                    
                    # Update the deployment with new image
                    kubectl set image deployment/${APP_NAME} ${APP_NAME}=${DOCKER_IMAGE} -n ${K8S_NAMESPACE} --record
                    
                    # Wait for rollout to complete
                    kubectl rollout status deployment/${APP_NAME} -n ${K8S_NAMESPACE} --timeout=120s
                    """
                }
            }
        }
        
        stage('Run Migrations') {
            steps {
                script {
                    // Run database migrations
                    sh """
                    kubectl exec deployment/${APP_NAME} -n ${K8S_NAMESPACE} -- \
                    python manage.py migrate --noinput
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    // Get the service URL and test
                    sh """
                    # Get service URL
                    SERVICE_URL=\$(minikube service ${APP_NAME}-service -n ${K8S_NAMESPACE} --url)
                    echo "Service URL: \$SERVICE_URL"
                    
                    # Test health endpoint
                    curl -f \$SERVICE_URL/health/ || echo "Health check failed, but continuing..."
                    """
                }
            }
        }
        
        stage('Show Application Info') {
            steps {
                script {
                    sh """
                    echo "=== Deployment Status ==="
                    kubectl get pods -n ${K8S_NAMESPACE}
                    
                    echo "=== Service Information ==="
                    kubectl get svc -n ${K8S_NAMESPACE}
                    
                    echo "=== Application URL ==="
                    minikube service ${APP_NAME}-service -n ${K8S_NAMESPACE} --url
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo '✅ Deployment completed successfully!'
            script {
                def appUrl = sh(script: "minikube service ${APP_NAME}-service -n ${K8S_NAMESPACE} --url", returnStdout: true).trim()
                echo "Application is available at: ${appUrl}"
            }
        }
        failure {
            echo '❌ Deployment failed!'
            script {
                // Show logs for debugging
                sh """
                kubectl describe deployment/${APP_NAME} -n ${K8S_NAMESPACE} || true
                kubectl logs deployment/${APP_NAME} -n ${K8S_NAMESPACE} --tail=50 || true
                """
            }
        }
        always {
            echo 'Pipeline execution completed'
            // Cleanup old images (optional)
            sh "docker image prune -f || true"
        }
    }
}