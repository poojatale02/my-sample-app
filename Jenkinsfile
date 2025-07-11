**The full updated `Jenkinsfile` will look like this:**

```groovy
// Jenkinsfile for Docker Build, Push, and Deploy
pipeline {
    agent any // Or specify a Docker agent if you have one configured

    environment {
        // Replace with your Docker Hub username
        DOCKER_HUB_USERNAME = 'your_docker_hub_username' // <<< REMEMBER TO UPDATE THIS
        IMAGE_NAME = 'my-sample-app'

        // For ECR (uncomment and update if you decide to use ECR instead of Docker Hub)
        // AWS_ACCOUNT_ID = '123456789012' // <<< REPLACE WITH YOUR AWS ACCOUNT ID
        // AWS_REGION = 'us-west-1' // <<< REPLACE WITH YOUR AWS REGION
        // ECR_REPO_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${IMAGE_NAME}"

        APP_SERVER_IP = '10.0.4.231' // <<< REPLACE WITH YOUR APP SERVER PRIVATE IP
        SSH_KEY_CREDENTIAL_ID = 'jenkins-ssh-key' // ID of the SSH key credential in Jenkins
        DOCKER_HUB_CREDENTIAL_ID = 'dockerhub-credentials' // ID of the Docker Hub credential in Jenkins
    }

    // Prevent Jenkins from doing its own implicit checkout
    options {
        skipDefaultCheckout()
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                // This step performs the SCM checkout configured in the job
              checkout scm
            }
      }

        stage('Clean Workspace') {
            steps {
                cleanWs()
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
                    sh "echo ${credentials(env.DOCKER_HUB_CREDENTIAL_ID).password} | /usr/bin/docker login --username ${env.DOCKER_HUB_USERNAME} --password-stdin"
                    sh "/usr/bin/docker push ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}"
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
                    sh """
                        ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ubuntu@${env.APP_SERVER_IP} <<EOF
                            echo ${credentials(env.DOCKER_HUB_CREDENTIAL_ID).password} | /usr/bin/docker login --username ${env.DOCKER_HUB_USERNAME} --password-stdin
                            /usr/bin/docker stop ${env.IMAGE_NAME} || true
                            /usr/bin/docker rm ${env.IMAGE_NAME} || true
                            /usr/bin/docker pull ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}
                            /usr/bin/docker run -d --name ${env.IMAGE_NAME} -p 80:80 ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}
                        EOF
                    """
                }
            }
        }
    }
}

