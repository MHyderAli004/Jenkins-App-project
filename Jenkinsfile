pipeline {
    agent any

    environment {
        // --- Jenkins Credential IDs (Manage Jenkins -> Credentials) ---
        DOCKER_CREDS_ID    = 'docker-hub-credentials'       // Kind: Username with password (Docker Hub)
        KUBECONFIG_CRED_ID = 'kubeconfig-file-credentials'  // Kind: Secret file (your kubeconfig)

        // --- Docker Hub image names ---
        DOCKER_HUB_USER = 'mhyderali004'
        BACKEND_IMAGE   = "${DOCKER_HUB_USER}/cicd-backend"
        FRONTEND_IMAGE  = "${DOCKER_HUB_USER}/cicd-frontend"
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Checking out source code from GitHub...'
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
                    echo 'Packaging the backend into a deployable JAR...'
                    sh 'mvn package -DskipTests'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                echo 'Building Docker images for frontend and backend...'
                sh 'docker compose build'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo 'Logging into Docker Hub and pushing images...'
                withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDS_ID,
                                                    usernameVariable: 'DOCKER_USER',
                                                    passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh 'docker compose push'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying updated images to the Kubernetes cluster...'

                // Injects the kubeconfig secret file into the KUBECONFIG env var.
                // kubectl reads KUBECONFIG automatically - no plugin required.
                withCredentials([file(credentialsId: env.KUBECONFIG_CRED_ID,
                                      variable: 'KUBECONFIG')]) {

                    // Apply all manifests from the k8s/ folder
                    sh 'kubectl apply -f k8s/'

                    // Force pods to restart so they pull the newly pushed ':latest' images
                    sh 'kubectl rollout restart deployment mysql'
                    sh 'kubectl rollout restart deployment backend'
                    sh 'kubectl rollout restart deployment frontend'

                    // Wait until each deployment is stable (fails the build if pods crash)
                    sh 'kubectl rollout status deployment/mysql    --timeout=120s'
                    sh 'kubectl rollout status deployment/backend  --timeout=120s'
                    sh 'kubectl rollout status deployment/frontend --timeout=120s'
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline succeeded! Application is live on Kubernetes.'
        }
        failure {
            echo '❌ Pipeline failed. Check the logs above.'
        }
        always {
            echo 'Cleaning up Docker artifacts on the Jenkins agent...'
            sh 'docker logout || true'
            sh 'docker image prune -f || true'
        }
    }
}
