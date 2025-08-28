pipeline {
    agent any

    environment {
        IMAGE_NAME = 'sample-node-app'
        IMAGE_TAG = 'v1.0'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/skmudabbir/sample-node-app.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                  set "HTTP_PROXY="
                  set "HTTPS_PROXY="
                  set "NO_PROXY=localhost,127.0.0.1,::1,kubernetes.docker.internal,*.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
                    
                    bat 'kubectl apply -f k8s-deployment.yaml'
                    bat 'kubectl apply -f k8s-service.yaml'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
    }
}





