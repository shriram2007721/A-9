pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = "shriram1106/lms-portal"
        VERSION         = "1.0.${BUILD_NUMBER}"
        DOCKER_CREDS    = credentials('docker-hub-credentials')
    }
    
    stages {
        stage('1. Clone Repository') {
            steps {
                checkout scm
            }
        }
        
        stage('2. Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_REGISTRY}:${VERSION} ."
                sh "docker tag ${DOCKER_REGISTRY}:${VERSION} ${DOCKER_REGISTRY}:latest"
            }
        }
        
        stage('3. Push to Docker Hub') {
            steps {
                sh "echo '${DOCKER_CREDS_PSW}' | docker login -u '${DOCKER_CREDS_USR}' --password-stdin"
                sh "docker push ${DOCKER_REGISTRY}:${VERSION}"
                sh "docker push ${DOCKER_REGISTRY}:latest"
            }
        }
        
        stage('4. Deploy to Kubernetes') {
            steps {
                // Dynamically inject the tagged version release into deployment spec
                sh "sed -i 's|shriram1106/lms-portal:latest|${DOCKER_REGISTRY}:${VERSION}|g' deployment.yaml"
                sh "kubectl apply -f deployment.yaml"
            }
        }
        
        stage('5. Verify Rollout & Health Check') {
            steps {
                script {
                    try {
                        // Monitor the rollout for up to 60 seconds
                        sh "kubectl rollout status deployment/lms-deployment --timeout=60s"
                        echo "Deployment health checks passed successfully!"
                    } catch (Exception e) {
                        echo "Deployment health check failed! Activating automated rollback..."
                        sh "kubectl rollout undo deployment/lms-deployment"
                        sh "kubectl rollout status deployment/lms-deployment"
                        error("Pipeline halted and rolled back due to unstable deployment version.")
                    }
                }
            }
        }
        
        stage('6. Print Verification States') {
            steps {
                echo "=== REPLICAS AND PODS RUNNING STATE ==="
                sh "kubectl get pods -l app=lms"
                echo "=== SERVICE NODEPORT ACCESS ==="
                sh "kubectl get svc lms-service"
            }
        }
    }
}
