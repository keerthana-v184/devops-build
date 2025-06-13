pipeline {
    agent { label 'node-react-agent' }

    environment {
        DOCKER_REGISTRY = "docker.io"
        DEV_IMAGE = "iamkeerthana/dev:dev"
        PROD_IMAGE = "iamkeerthana/prod:prod"
        DOCKER_CREDS = "docker-hub-credentials"
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Setup Image Tag') {
            steps {
                script {
                    def branch = env.GIT_BRANCH?.replace('origin/', '') ?: sh(
                        script: "git rev-parse --abbrev-ref HEAD",
                        returnStdout: true
                    ).trim()

                    if (branch == 'dev') {
                        env.DOCKER_IMAGE = "${DOCKER_REGISTRY}/${DEV_IMAGE}"
                    } else if (branch == 'main') {
                        env.DOCKER_IMAGE = "${DOCKER_REGISTRY}/${PROD_IMAGE}"
                    } else {
                        error("Unsupported branch: ${branch}")
                    }

                    echo "Using Docker Image: ${env.DOCKER_IMAGE}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${env.DOCKER_IMAGE} ."
                sh "docker image inspect ${env.DOCKER_IMAGE} > /dev/null"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKER_CREDS,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}
                    '''
                }
            }
        }

        stage('Deploy App on Node') {
            steps {
                sh '''
                    docker pull ${DOCKER_IMAGE}
                    docker stop react-app || true
                    docker rm react-app || true
                    docker run -d --name react-app -p 80:80 ${DOCKER_IMAGE}
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            sh "docker rmi ${env.DOCKER_IMAGE} || true"
            cleanWs()
        }
        success {
            echo "Successfully deployed ${env.DOCKER_IMAGE} on node-react-agent"
        }
        failure {
            echo "Deployment failed for ${env.DOCKER_IMAGE}"
        }
    }
}

