pipeline {
    agent any
    
    // NEW: Tool directive added here to load Maven 'M3' onto the PATH
    tools {
        maven 'M3' 
    }

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
        stage('Checkout') {
            steps {
                // CORRECTED: Using your GitHub ID 'github-credentials'
                git branch: 'main', credentialsId: 'github-credentials', url: 'https://github.com/tech-911/Devops-petcliinc.git'
            }
        }

        stage('Build & Test') {
            steps {
                // Task 3: Build the application and run tests
                sh 'mvn clean install -DskipTests' // Compile and build JAR
                sh 'mvn test' // Run unit tests (Lab requirement)
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Builds the image using the Dockerfile located in the repository root
                    docker.build(DOCKER_IMAGE)
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                script {
                    // CORRECTED: Using your Docker Hub ID 'dockerhub-credentials'
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin"
                        sh "docker push ${DOCKER_IMAGE}"
                        // Also tag and push as 'latest' for easy deployment updates
                        sh "docker tag ${DOCKER_IMAGE} ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
                        sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // Task 4: Apply Kubernetes manifests
                // Assumes your k8s files (deployment.yml, service.yml) are in a subfolder named 'k8s'
                // This 'sed' command updates the image tag in your deployment file before applying it
                sh "sed -i 's|bolatunj/devops-petcliinc:latest|${DOCKER_IMAGE}|g' k8s/deployment.yml"
                sh "kubectl apply -f k8s/service.yml"
                sh "kubectl apply -f k8s/deployment.yml"
                // Task 5: Check rollout status
                sh "kubectl rollout status deployment/${K8S_DEPLOYMENT_NAME} --namespace default"
            }
        }
    }
}
