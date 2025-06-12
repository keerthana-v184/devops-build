pipeline {
    agent any

    environment {
        DOCKER_DEV_REPO = "iamkeerthana/dev"
        DOCKER_PROD_REPO = "iamkeerthana/prod"
        DOCKER_REGISTRY = "docker.io"
        AGENT_IP = "35.173.187.48"
        AGENT_SSH_CREDS = "ssh-credentials"  // Your SSH credentials ID
        DOCKER_HUB_CREDS = "docker-hub-credentials"  // Your Docker Hub credentials ID
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                script {
                    env.BRANCH_NAME = env.GIT_BRANCH?.replace("origin/", "") ?: sh(
                        script: "git rev-parse --abbrev-ref HEAD",
                        returnStdout: true
                    ).trim()

                    if (env.BRANCH_NAME == 'dev') {
                        env.DOCKER_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_DEV_REPO}:dev"
                    } else if (env.BRANCH_NAME == 'main') {
                        env.DOCKER_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_PROD_REPO}:prod"
                    } else {
                        error("Unsupported branch '${env.BRANCH_NAME}'. Only 'dev' and 'main' are allowed.")
                    }

                    echo "Building and deploying image: ${env.DOCKER_IMAGE}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        docker build -t ${env.DOCKER_IMAGE} .
                        docker images | grep iamkeerthana
                    """
                    
                    // Verify image built successfully
                    if (sh(script: "docker inspect ${env.DOCKER_IMAGE}", returnStatus: true) != 0) {
                        error("Failed to build Docker image")
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKER_HUB_CREDS,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    script {
                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker push ${env.DOCKER_IMAGE}
                        """
                    }
                }
            }
        }

        stage('Deploy to Node-React Server') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: env.AGENT_SSH_CREDS,
                        usernameVariable: 'SSH_USER',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no -i \$SSH_KEY \$SSH_USER@${env.AGENT_IP} \"
                                docker pull ${env.DOCKER_IMAGE}
                                docker stop react-app || true
                                docker rm react-app || true
                                docker run -d --name react-app -p 80:80 ${env.DOCKER_IMAGE}
                            \"
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                sh 'docker logout || true'
                sh "docker rmi ${env.DOCKER_IMAGE} || true"
                cleanWs()
            }
        }
        success {
            echo "Successfully completed pipeline for ${env.BRANCH_NAME}"
            echo "Deployed image: ${env.DOCKER_IMAGE}"
        }
        failure {
            echo "Pipeline failed for branch ${env.BRANCH_NAME}"
            slackSend(color: 'danger', message: "Pipeline Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}")
        }
    }
}
