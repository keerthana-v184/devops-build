pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('docker-hub-credentials')
        GITHUB_CREDENTIALS = credentials('github-credentials')
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'dev', credentialsId: 'github-credentials', url: 'https://github.com/keerthana-v184/devops-build.git'
            }
        }
        stage('Build') {
            steps {
                sh './build.sh'
            }
        }
        stage('Push to Docker Hub') {
            when {
                branch 'dev'
            }
            steps {
                sh 'docker tag react-app iamkeerthana/dev:dev'
                sh 'docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}'
                sh 'docker push iamkeerthana/dev:dev'
            }
        }
        stage('Push to Prod') {
            when {
                branch 'main'
            }
            steps {
                sh 'docker tag react-app iamkeerthana/prod:prod'
                sh 'docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}'
                sh 'docker push iamkeerthana/prod:prod'
            }
        }
        stage('Deploy') {
            agent {
                label 'node-react'  // Deploys on your Node-React agent
            }
            steps {
                sh '''
                    cd /home/ubuntu/app  # Actual path to your project
                    docker-compose pull
                    docker-compose up -d
                '''
            }
        }
    }
}
