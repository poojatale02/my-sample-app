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

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    // Build the Docker image using the full path to the docker executable
                    // The '.' at the end means build from the current directory (where Jenkinsfile is)
                    sh "/usr/bin/docker build -t ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER} ."
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    // Authenticate to Docker Hub using the credentials stored in Jenkins
                    // Access password using credentials() binding
                    sh "echo ${credentials(env.DOCKER_HUB_CREDENTIAL_ID).password} | /usr/bin/docker login --username ${env.DOCKER_HUB_USERNAME} --password-stdin"
                    
                    // Push the Docker image to Docker Hub using the full path
                    sh "/usr/bin/docker push ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}"

                    // For ECR (uncomment and update if you decide to use ECR)
                    // sh "aws ecr get-login-password --region ${env.AWS_REGION} | /usr/bin/docker login --username AWS --password-stdin ${env.ECR_REPO_URI}"
                    // sh "/usr/bin/docker push ${env.ECR_REPO_URI}:${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Docker Cleanup (Local)') {
            steps {
                script {
                    // Remove the local Docker image after push to save space on the Jenkins agent
                    sh "/usr/bin/docker rmi ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Deploy to App Machine') {
            steps {
                // Use SSH Agent plugin for secure key usage
                // Ensure 'jenkins-ssh-key' is a 'SSH Username with Private Key' credential in Jenkins
                // with your id_rsa_terraform private key.
                sshagent(credentials: [env.SSH_KEY_CREDENTIAL_ID]) {
                    sh """
                        # SSH into the App Server from Jenkins agent (assuming agent has direct access or via bastion)
                        # -o StrictHostKeyChecking=no: Disables strict host key checking (useful for dynamic IPs)
                        # -o UserKnownHostsFile=/dev/null: Prevents adding host keys to known_hosts
                        ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ubuntu@${env.APP_SERVER_IP} <<EOF
                            # Login to Docker Hub on the app machine (if needed for private repos)
                            # This is important if your Docker Hub image is private
                            echo ${credentials(env.DOCKER_HUB_CREDENTIAL_ID).password} | /usr/bin/docker login --username ${env.DOCKER_HUB_USERNAME} --password-stdin

                            # Stop and remove any old container of the same name
                            # '|| true' prevents the script from failing if the container doesn't exist
                            /usr/bin/docker stop ${env.IMAGE_NAME} || true
                            /usr/bin/docker rm ${env.IMAGE_NAME} || true

                            # Pull the new image from Docker Hub
                            /usr/bin/docker pull ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}

                            # Run the new container in detached mode (-d)
                            # --name: Assigns a name to the container
                            # -p 80:80: Maps host port 80 to container port 80 (Nginx default)
                            /usr/bin/docker run -d --name ${env.IMAGE_NAME} -p 80:80 ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}
                        EOF
                    """
                }
            }
        }
    }
}

