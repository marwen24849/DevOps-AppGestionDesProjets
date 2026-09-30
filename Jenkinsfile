pipeline {
    agent any

    tools {
        jdk 'JDK17'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_NAMESPACE = 'replace-with-your-dockerhub-username'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_CREDENTIALS_ID = 'docker-registry-credentials'
        SONARQUBE_SERVER = 'SonarQube'
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

        stage('Analyse avec SonarQube') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    dir('backend') {
                        sh './mvnw -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=gestion-projets'
                    }
                }
            }
        }

        stage('Tests unitaires') {
            steps {
                sh '''
                    docker compose up -d mysql
                    ready=0
                    for attempt in $(seq 1 30); do
                        if docker compose exec -T mysql mysqladmin ping -h localhost -uroot -proot --silent; then
                            ready=1
                            break
                        fi
                        sleep 2
                    done
                    test "$ready" -eq 1
                '''
                dir('backend') {
                    sh '''
                        SPRING_DATASOURCE_URL='jdbc:mysql://127.0.0.1:3307/test_db?createDatabaseIfNotExist=true' \
                        SPRING_DATASOURCE_USERNAME=root \
                        SPRING_DATASOURCE_PASSWORD=root \
                        ./mvnw -B test
                    '''
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

    post {
        always {
            junit allowEmptyResults: true, testResults: 'backend/target/surefire-reports/*.xml'
        }
    }
}