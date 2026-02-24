pipeline {
    agent any

    stages {

        stage('Clone Code') {
            steps {
                git branch: 'main', 
                    credentialsId: 'github-credentials', 
                    url: 'https://github.com/cx4167/crud-dd-task-mean-app.git'
            }
        }

        stage('Build Backend') {
            steps {
                sh 'docker build -t denish952/mean-backend:latest ./backend'
            }
        }

        stage('Build Frontend') {
            steps {
                sh 'docker build -t denish952/mean-frontend:latest ./frontend'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    sh 'echo $PASSWORD | docker login -u $USERNAME --password-stdin'
                    sh 'docker push denish952/mean-backend:latest'
                    sh 'docker push denish952/mean-frontend:latest'
                }
            }
        }

        stage('Deploy App') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d'
            }
        }

    }

    post {
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Something went wrong. Check the logs.'
        }
    }
}
