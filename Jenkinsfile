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
        // DOCKER_HUB_CREDENTIAL_ID is commented out as we're hardcoding it for debugging
        // DOCKER_HUB_CREDENTIAL_ID = 'dockerhub-credentials'
    }

    // Prevent Jenkins from doing its own implicit checkout.
    // We will perform an explicit checkout in the 'Checkout Source Code' stage.
    options {
        skipDefaultCheckout()
    }

    stages {
        stage('Clean Workspace') { // Moved to be the first stage
            steps {
                cleanWs() // Cleans the workspace directory before starting the build.
            }
        }

        stage('Checkout Source Code') { // Moved to be the second stage
            steps {
                // This step performs the SCM checkout configured in the Jenkins job.
                // It will pull the repository content into the workspace.
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    // Builds the Docker image.
                    // Using '/usr/bin/docker' explicitly to ensure the command is found inside the Jenkins container.
                    // The '.' at the end indicates the build context is the current directory (the workspace).
                    sh "/usr/bin/docker build -t ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER} ."
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    // Use withCredentials to securely access Docker Hub password.
                    // Hardcoding credentialsId for debugging to rule out variable resolution issues.
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        // Authenticates to Docker Hub using the credentials.
                        // The '$' before DOCKER_PASS and DOCKER_USER ensures the shell interprets them as variables.
                        sh "echo \$DOCKER_PASS | /usr/bin/docker login --username \$DOCKER_USER --password-stdin"
                        
                        // Pushes the built Docker image to Docker Hub.
                        sh "/usr/bin/docker push ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}"
                    }

                    // For ECR (uncomment and update if you decide to use ECR instead of Docker Hub):
                    // sh "aws ecr get-login-password --region ${env.AWS_REGION} | /usr/bin/docker login --username AWS --password-stdin ${env.ECR_REPO_URI}"
                    // sh "/usr/bin/docker push ${env.ECR_REPO_URI}:${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Docker Cleanup (Local)') {
            steps {
                script {
                    // Removes the Docker image from the local Jenkins agent's Docker daemon
                    // after it has been successfully pushed to the registry.
                    sh "/usr/bin/docker rmi ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Deploy to App Machine') {
            steps {
                // Uses the SSH Agent plugin to securely provide the SSH private key
                // for connecting to the App Server.
                sshagent(credentials: [env.SSH_KEY_CREDENTIAL_ID]) {
                    // Using << "EOF" to prevent Groovy interpolation within the heredoc.
                    // This allows shell variables (like $DOCKER_USER_APP, $DOCKER_PASS_APP) to be used.
                    sh """
                        # SSH into the App Server from the Jenkins agent.
                        # -o StrictHostKeyChecking=no: Disables strict host key checking (useful for dynamic IPs).
                        # -o UserKnownHostsFile=/dev/null: Prevents adding host keys to known_hosts.
                        ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ubuntu@${env.APP_SERVER_IP} << "EOF"
                            // Use withCredentials inside the SSH session for Docker Hub login on the app machine.
                            // Hardcoding credentialsId for debugging to rule out variable resolution issues.
                            withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER_APP', passwordVariable: 'DOCKER_PASS_APP')]) {
                                # Login to Docker Hub on the App Server (necessary if your Docker Hub image is private).
                                sh "echo \$DOCKER_PASS_APP | /usr/bin/docker login --username \$DOCKER_USER_APP --password-stdin"
                            }

                            # Stop and remove any old container of the same name.
                            # '|| true' prevents the script from failing if the container doesn't exist.
                            /usr/bin/docker stop ${env.IMAGE_NAME} || true
                            /usr/bin/docker rm ${env.IMAGE_NAME} || true

                            # Pull the new image from Docker Hub to the App Server.
                            /usr/bin/docker pull ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}

                            # Run the new container in detached mode (-d).
                            # --name: Assigns a specific name to the container.
                            # -p 80:80: Maps host port 80 to container port 80 (Nginx default).
                            /usr/bin/docker run -d --name ${env.IMAGE_NAME} -p 80:80 ${env.DOCKER_HUB_USERNAME}/${env.IMAGE_NAME}:${env.BUILD_NUMBER}
                        EOF
                    """
                }
            }
        }
    }
}

