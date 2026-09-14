pipeline {
    agent any

    environment {
        // --- CREDENTIALS IDS (Must match what you configured in Jenkins UI) ---
        DOCKER_CREDS_ID = 'docker-hub-credentials'
        KUBECONFIG_CRED_ID = 'kubeconfig-file-credentials'
        
        // --- DOCKER HUB DETAILS ---
        DOCKER_HUB_USER = 'mhyderali004'
        BACKEND_IMAGE = "${DOCKER_HUB_USER}/cicd-backend"
        FRONTEND_IMAGE = "${DOCKER_HUB_USER}/cicd-frontend"
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Checking out source code from GitHub...'
                // Checks out the repository based on the Jenkins job configuration
                checkout scm 
            }
        }

        stage('Build Backend') {
            steps {
                dir('backend') {
                    echo 'Compiling the backend Java application...'
                    sh 'mvn clean compile'
                }
            }
        }

        stage('Run Tests') {
            steps {
                dir('backend') {
                    echo 'Running automated backend unit tests...'
                    sh 'mvn test'
                }
            }
        }

        stage('Package Application') {
            steps {
                dir('backend') {
                    echo 'Packaging the backend application into a deployable JAR...'
                    // Skip tests here as they already passed in the previous stage
                    sh 'mvn package -DskipTests' 
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                echo 'Building Docker images for frontend and backend via Docker Compose...'
                // This reads your docker-compose.yml and builds the images
                sh 'docker compose build'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo 'Logging into Docker Hub and pushing new images...'
                withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    
                    // Pushes the images defined in docker-compose.yml to Docker Hub
                    sh 'docker compose push'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying updated images to the Kubernetes cluster...'
                
                // The 'withKubeConfig' step is provided by the Kubernetes CLI plugin.
                // It securely loads your cluster credentials for the duration of this block.
                withKubeConfig([credentialsId: env.KUBECONFIG_CRED_ID]) {
                    
                    // 1. Apply the K8s manifests located in the k8s/ directory
                    echo 'Applying Kubernetes manifests...'
                    sh 'kubectl apply -f k8s/'
                    
                    // 2. Force a rollout restart. 
                    // Because we use the ':latest' tag, K8s won't pull the new image automatically 
                    // unless we force a restart of the pods.
                    echo 'Restarting deployments to pull new images...'
                    sh 'kubectl rollout restart deployment mysql'
                    sh 'kubectl rollout restart deployment backend'
                    sh 'kubectl rollout restart deployment frontend'
                    
                    // 3. Wait for the deployments to stabilize (ensures pods are actually running)
                    echo 'Waiting for deployments to stabilize...'
                    sh 'kubectl rollout status deployment/mysql --timeout=120s'
                    sh 'kubectl rollout status deployment/backend --timeout=120s'
                    sh 'kubectl rollout status deployment/frontend --timeout=120s'
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline executed successfully! Application is live on Kubernetes.'
            // You can add Slack or Email notifications here
            // slackSend channel: '#deployments', message: "Deployment successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        failure {
            echo '❌ Pipeline failed. Please check the logs.'
            // slackSend channel: '#deployments', color: 'danger', message: "Deployment failed: ${env.JOB_NAME}"
        }
        always {
            echo 'Cleaning up Docker workspace to save agent disk space...'
            sh 'docker logout'
            // Remove dangling images and build cache
            sh 'docker image prune -f'
            sh 'docker builder prune -f'
        }
    }
}
