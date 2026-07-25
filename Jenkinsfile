pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
        IMAGE_NAME = 'jayanth1542/hello-cicd'
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Test') {
            steps { sh 'npm install && npm test' }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh "echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin"
                sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
                sh "docker push ${IMAGE_NAME}:latest"
            }
        }

        stage('Deploy with Ansible') {
            steps {
                sh "ansible-playbook -i inventory.ini deploy.yml -e image_tag=${BUILD_NUMBER}"
            }
        }
    }

    post {
        success { echo "Build ${BUILD_NUMBER} deployed. Check: curl http://10.90.62.225:8081" }
        failure { echo "Pipeline failed at build ${BUILD_NUMBER}" }
    }
}
