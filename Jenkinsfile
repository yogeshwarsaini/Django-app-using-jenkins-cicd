 pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = "yogismash"
        DOCKER_IMAGE = "${DOCKERHUB_USERNAME}/django-"app
        CONTAINER_NAME = "django_container"

        APP_PORT = "8000"
        HOST_PORT = "8002"

        DOCKERFILE_PATH = "."
        HEALTH_CHECK_PATH = "/health"
        WAIT_TIME = "25"
        IMAGE_TAG = "${GIT_COMMIT}"
    }

    stages {

        stage('Cleanup') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build \
                -t ${DOCKER_IMAGE}:${IMAGE_TAG} \
                -t ${DOCKER_IMAGE}:latest \
                ${DOCKERFILE_PATH}
                """
            }
        }

        stage('DockerHub Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                }
            }
        }

        stage('Push Image') {
            steps {
                sh """
                docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                docker push ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Stop Old Container') {
            steps {
                sh '''
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh """
                docker run -d \
                --name ${CONTAINER_NAME} \
                -p ${HOST_PORT}:${APP_PORT} \
                ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Verify') {
            steps {
                sh "sleep ${WAIT_TIME}"
                sh "curl -f http://localhost:${HOST_PORT}${HEALTH_CHECK_PATH}"
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
        }
    }
}
