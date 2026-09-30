pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_NAMESPACE = 'marwen24849' 
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_CREDENTIALS_ID = 'docker-hub-credentials' 
    }

    stages {
        stage('Get Code From Git') {
            steps {
                checkout scm
            }
        }

        stage('Compile') {
            steps {
                dir('backend') {
                    sh './mvnw -B compile'
                }
            }
        }

        stage('Generation du livrable') {
            steps {
                dir('backend') {
                    sh './mvnw -B package -DskipTests'
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Build des images Docker') {
            steps {
                sh '''
                    docker build -t "$DOCKER_REGISTRY/$DOCKER_NAMESPACE/projet-backend:$IMAGE_TAG" -f backend/Dockerfile backend
                    docker build -t "$DOCKER_REGISTRY/$DOCKER_NAMESPACE/projet-frontend:$IMAGE_TAG" -f frontend/Dockerfile frontend
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS_ID}",
                    usernameVariable: 'REGISTRY_USER',
                    passwordVariable: 'REGISTRY_PASSWORD'
                )]) {
                    sh '''
                        echo "$REGISTRY_PASSWORD" | docker login "$DOCKER_REGISTRY" --username "$REGISTRY_USER" --password-stdin
                        docker push "$DOCKER_REGISTRY/$DOCKER_NAMESPACE/projet-backend:$IMAGE_TAG"
                        docker push "$DOCKER_REGISTRY/$DOCKER_NAMESPACE/projet-frontend:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('Docker Compose up -d') {
            steps {
                sh '''
                    docker compose pull backend frontend
                    docker compose up -d --no-build
                    docker compose ps
                '''
            }
        }
    }
}

