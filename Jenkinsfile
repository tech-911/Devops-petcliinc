pipeline {
    agent none // No global agent. Stages define their own agents.

    environment {
        // Define Docker Image variables using your usernames
        DOCKER_HUB_USER = 'bolatunj'
        IMAGE_NAME = "devops-petcliinc"
        // Use the Jenkins Build Number for a unique tag (Task 3 requirement)
        IMAGE_TAG = "${env.BUILD_NUMBER}" 
        DOCKER_IMAGE = "${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
        // Define Kubernetes deployment name (Assuming 'petclinic-deployment' from your manifest)
        K8S_DEPLOYMENT_NAME = "petclinic-deployment"
    }

    stages {
        // Stage 1: Checkout (Runs on any available agent)
        stage('Checkout') {
            agent any 
            steps {
                git branch: 'main', credentialsId: 'github-credentials', url: 'https://github.com/tech-911/Devops-petcliinc.git'
            }
        }

        // Stage 2: Build & Test 
        stage('Build & Test') {
            // Runs the steps inside a container with Maven (M3) and JDK 21 pre-installed.
            // This bypasses the Jenkins tool installer issue.
            agent {
                docker {
                    // Use a standard Maven image built on a recent JDK (21 is currently very high)
                    image 'maven:3-openjdk-21' 
                    // Ensures the container uses the workspace created by the Checkout stage
                    reuseNode true 
                }
            }
            steps {
                // mvn is now available inside the Docker container
                sh 'mvn clean install -DskipTests' 
                sh 'mvn test'
            }
        }

        // Stage 3: Build Docker Image (Runs on any available agent with Docker access)
        stage('Build Docker Image') {
            agent any
            steps {
                script {
                    docker.build(DOCKER_IMAGE)
                }
            }
        }

        // Stage 4: Push Image to Docker Hub (Runs on any available agent)
        stage('Push Image to Docker Hub') {
            agent any
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin"
                        sh "docker push ${DOCKER_IMAGE}"
                        sh "docker tag ${DOCKER_IMAGE} ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
                        sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        // Stage 5: Deploy to Kubernetes (Runs on any available agent)
        stage('Deploy to Kubernetes') {
            agent any
            steps {
                sh "sed -i 's|bolatunj/devops-petcliinc:latest|${DOCKER_IMAGE}|g' k8s/deployment.yml"
                sh "kubectl apply -f k8s/service.yml"
                sh "kubectl apply -f k8s/deployment.yml"
                sh "kubectl rollout status deployment/${K8S_DEPLOYMENT_NAME} --namespace default"
            }
        }
    }
}
