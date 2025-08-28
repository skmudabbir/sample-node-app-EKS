pipeline {
    agent any

    environment {
        IMAGE_NAME = 'sample-node-app'
        IMAGE_TAG = 'v1.0'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                git 'https://github.com/skmudabbir/sample-node-app.git'
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
                    sh 'kubectl apply -f k8s-deployment.yaml'
                    sh 'kubectl apply -f k8s-service.yaml'
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

