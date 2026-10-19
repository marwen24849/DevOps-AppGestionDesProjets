pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_NAMESPACE = 'marwen695' 
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_CREDENTIALS_ID = 'docker-hub-credentials' 
        KUBECONFIG_CREDENTIALS_ID = 'kubeconfig'
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
                    // Force l'attribution des droits d'exécution sur le wrapper Maven
                    sh 'chmod +x ./mvnw'
                    sh './mvnw -B compile'
                }
            }
        }

        stage('Generation du livrable') {
            steps {
                dir('backend') {
                    // Sécurité additionnelle pour l'étape de packaging
                    sh 'chmod +x ./mvnw'
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

        stage('Deployment Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: "${KUBECONFIG_CREDENTIALS_ID}",
                    variable: 'KUBECONFIG'
                )]) {
                    sh '''
                        set -eu

                        kubectl apply -f k8s/mysql.yaml
                        kubectl rollout status deployment/mysql --timeout=180s

                        sed "s|docker.io/marwen695/projet-backend:latest|$DOCKER_REGISTRY/$DOCKER_NAMESPACE/projet-backend:$IMAGE_TAG|g" k8s/backend.yaml | kubectl apply -f -
                        sed "s|docker.io/marwen695/projet-frontend:latest|$DOCKER_REGISTRY/$DOCKER_NAMESPACE/projet-frontend:$IMAGE_TAG|g" k8s/frontend.yaml | kubectl apply -f -

                        kubectl rollout status deployment/projet-backend --timeout=180s
                        kubectl rollout status deployment/projet-frontend --timeout=180s
                        kubectl get services
                    '''
                }
            }
        }
    }
}
