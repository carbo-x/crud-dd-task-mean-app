pipeline {
    agent any
    environment {
        DOCKERHUB_USER = 'denish136'
        VERSION        = "v${BUILD_NUMBER}"
        BACKEND_IMAGE  = "${DOCKERHUB_USER}/mean-backend:${VERSION}"
        FRONTEND_IMAGE = "${DOCKERHUB_USER}/mean-frontend:${VERSION}"
    }
    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-credentials',
                    url: 'https://github.com/carbo-x/crud-dd-task-mean-app.git'
            }
        }
        stage('Build Backend') {
            steps {
                sh 'docker build -t $BACKEND_IMAGE ./backend'
            }
        }
        stage('Build Frontend') {
            steps {
                sh 'docker build -t $FRONTEND_IMAGE ./frontend'
            }
        }
        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'PASSWORD'
                )]) {
                    sh 'echo $PASSWORD | docker login -u $USERNAME --password-stdin'
                    sh 'docker push $BACKEND_IMAGE'
                    sh 'docker push $FRONTEND_IMAGE'
                }
            }
        }
        stage('Deploy App') {
            steps {
                sh 'docker compose down || true'
                sh 'IMAGE_TAG=$VERSION docker compose up -d'
            }
        }
    }
    post {
        success {
            echo "Deployment successful! Version: ${VERSION}"
        }
        failure {
            echo 'Something went wrong. Check the logs.'
        }
    }
}
