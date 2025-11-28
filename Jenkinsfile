pipeline {
    agent any

    environment {
        DOCKERHUB_BACKEND = "ritikkumarsahu/dd-mean-backend"
        DOCKERHUB_FRONTEND = "ritikkumarsahu/dd-mean-frontend"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Backend Image') {
            steps {
                sh '''
                  echo "[INFO] Building backend image..."
                  docker build -t ${DOCKERHUB_BACKEND}:latest ./backend
                '''
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                  echo "[INFO] Building frontend image..."
                  docker build -t ${DOCKERHUB_FRONTEND}:latest ./frontend
                '''
            }
        }

        stage('Login to Docker Hub & Push Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                      echo "[INFO] Logging in to Docker Hub..."
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                      echo "[INFO] Pushing backend image..."
                      docker push ${DOCKERHUB_BACKEND}:latest

                      echo "[INFO] Pushing frontend image..."
                      docker push ${DOCKERHUB_FRONTEND}:latest

                      echo "[INFO] Docker logout"
                      docker logout || true
                    '''
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                sh '''
                  echo "[INFO] Pulling latest images and redeploying containers..."
                  docker compose pull
                  docker compose up -d --remove-orphans
                  docker image prune -f || true
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Deployment successful!'
        }
        failure {
            echo '❌ Pipeline failed. Please check the logs.'
        }
    }
}
