pipeline {

    agent any

    environment {
        DOCKERHUB_USERNAME = "scalersamarth"
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/go-application"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'go build -v ./...'
            }
        }

        stage('Test') {
            steps {
                sh 'go test -v ./...'
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                    -t ${IMAGE_NAME}:${IMAGE_TAG} \
                    -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Push') {

            when {
                not {
                    changeRequest()
                }
            }

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_PASS'
                    )
                ]) {

                    sh '''
                        echo "$DH_PASS" | docker login \
                        -u "$DH_USER" \
                        --password-stdin

                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy') {

            when {
                not {
                    changeRequest()
                }
            }

            agent {
                label 'executor-node'
            }

            steps {

                sh '''
                    docker pull ${IMAGE_NAME}:${IMAGE_TAG}

                    docker stop go-application || true
                    docker rm go-application || true

                    docker run -d \
                        --name go-application \
                        -p 8080:8080 \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }
    }
}
