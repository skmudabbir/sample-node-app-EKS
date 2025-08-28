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




