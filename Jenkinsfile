// Jenkinsfile for Docker Build, Push, and Deploy
pipeline {
    agent any

    environment {
        DOCKER_HUB_USERNAME = 'poojatale02'
        IMAGE_NAME = 'my-sample-app'
        APP_SERVER_IP = '10.0.4.231'
        SSH_KEY_CREDENTIAL_ID = 'jenkins-ssh-key'
    }

    options {
        skipDefaultCheckout()
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    sh "/usr/bin/docker build -t ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER} ."
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh "echo \$DOCKER_PASS | /usr/bin/docker login --username \$DOCKER_USER --password-stdin"
                        sh "/usr/bin/docker push ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('Docker Cleanup (Local)') {
            steps {
                script {
                    sh "/usr/bin/docker rmi ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Deploy to App Machine') {
            steps {
                sshagent(credentials: [env.SSH_KEY_CREDENTIAL_ID]) {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER_APP',
                        passwordVariable: 'DOCKER_PASS_APP'
                    )]) {
                        // Correct use of EOF with no indentation before it
                        sh """
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ubuntu@${env.APP_SERVER_IP} <<EOF
  echo \$DOCKER_PASS_APP | docker login --username \$DOCKER_USER_APP --password-stdin

  docker stop ${env.IMAGE_NAME} || true
  docker rm ${env.IMAGE_NAME} || true

  docker pull ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}

  docker run -d --name ${env.IMAGE_NAME} -p 80:80 ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}
EOF
                        """
                    }
                }
            }
        }
    }
}
