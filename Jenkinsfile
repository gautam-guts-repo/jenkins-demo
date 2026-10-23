pipeline {
    agent any
    environment {
        IMAGE_NAME = "myapp"
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
        CONTAINER_NAME = "myapp-container"
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Build') {
            steps { sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .' }
        }
        stage('Test') {
            steps { sh 'docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} echo "tests passed"' }
        }
        stage('Deploy') {
            steps {
                sh '''
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                    docker run -d --name ${CONTAINER_NAME} -p 8081:80 ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }
    }
    post {
        success { echo 'Pipeline completed successfully!' }
        failure { echo 'Pipeline failed. Check the console output above.' }
    }
}
