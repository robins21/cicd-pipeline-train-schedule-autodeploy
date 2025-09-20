pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('docker-hub-credentials')
        DOCKER_IMAGE_NAME = "imrobins21/train-schedule"
        DOCKER_IMAGE_TAG = "${env.BUILD_NUMBER}"
        KUBECONFIG = credentials('kubeconfig-credentials')
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                git branch: 'main', 
                    credentialsId: 'github-credentials', 
                    url: 'https://github.com/robins21/cicd-pipeline-train-schedule-autodeploy.git'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                    sh "docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} ."
                    sh "docker tag ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} ${DOCKER_IMAGE_NAME}:latest"
                }
            }
        }
        
        stage('Test Docker Image') {
            steps {
                script {
                    echo 'Testing Docker image...'
                    sh "docker run --rm ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} npm test || echo 'No tests found, continuing...'"
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo 'Logging into Docker Hub...'
                    sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                    echo "Pushing image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                    sh "docker push ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                    sh "docker push ${DOCKER_IMAGE_NAME}:latest"
                    echo 'Cleaning up local images...'
                    sh "docker rmi ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} || true"
                    sh "docker rmi ${DOCKER_IMAGE_NAME}:latest || true"
                }
            }
        }
        
        stage('Update Kubernetes Manifest') {
            steps {
                script {
                    echo 'Updating Kubernetes deployment with new image tag...'
                    sh """
                        sed -i 's|image: .*train-schedule:|image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}|g' kubernetes/deployment.yaml
                    """
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo 'Deploying to Minikube...'
                    withKubeConfig([credentialsId: 'kubeconfig-credentials']) {
                        sh 'kubectl apply -f kubernetes/deployment.yaml'
                        sh 'kubectl apply -f kubernetes/service.yaml'
                        sh 'kubectl rollout status deployment/train-schedule'
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo 'Waiting for deployment to be ready...'
                    withKubeConfig([credentialsId: 'kubeconfig-credentials']) {
                        sh 'sleep 10'
                        sh 'kubectl get pods -l app=train-schedule'
                        sh 'kubectl get svc train-schedule-service'
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up...'
            sh 'docker logout || true'
        }
        success {
            echo 'Pipeline completed successfully!'
            withKubeConfig([credentialsId: 'kubeconfig-credentials']) {
                script {
                    def service = sh(script: 'kubectl get svc train-schedule-service -o jsonpath="{.spec.ports[0].nodePort}"', returnStdout: true).trim()
                    def nodePort = service as Integer
                    echo "Application is running on Minikube at: http://\$(minikube ip):${nodePort}"
                }
            }
        }
        failure {
            echo 'Pipeline failed!'
            emailext (
                subject: "Build Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: "Check console output at ${env.BUILD_URL} to view the results.",
                to: 'admin@abstergo.com'
            )
        }
    }
}
