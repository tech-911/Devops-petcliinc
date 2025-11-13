pipeline {
    agent any // Initial agent, used for Checkout, Build & Test
    
    tools {
        jdk 'JDK-17' 
        maven 'M3' 
    }

    environment {
        DOCKER_HUB_USER = 'bolatunj'
        IMAGE_NAME = "devops-petcliinc"
        IMAGE_TAG = "${env.BUILD_NUMBER}" 
        DOCKER_IMAGE = "${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
        K8S_DEPLOYMENT_NAME = "petclinic-deployment"
    }

    stages {
        stage('Checkout') {
            // This runs successfully on the initial 'agent any'
            steps {
                git branch: 'main', credentialsId: 'github-credentials', url: 'https://github.com/tech-911/Devops-petcliinc.git'
            }
        }

        stage('Build & Test') {
            // This runs successfully on the initial 'agent any'
            steps {
                sh 'mvn clean install -DskipTests' 
                sh 'mvn test' 
            }
        }

        stage('Build & Push Docker Image (Kaniko)') {
            // Jenkins will spin up a new Kaniko Pod using the label defined in Step 2
            agent { label 'kaniko-builder' } 
            steps {
                script {
                    // 1. Get Docker Hub credentials securely
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials', 
                        passwordVariable: 'DOCKER_PASSWORD', 
                        usernameVariable: 'DOCKER_USERNAME')]) {
                        
                        // 2. Create the Docker Hub configuration file required by Kaniko
                        // This file is mounted directly into the Kaniko container's filesystem
                        sh """
                        mkdir -p /kaniko/.docker
                        cat > /kaniko/.docker/config.json <<EOF
                        {
                            "auths": {
                                "https://index.docker.io/v1/": {
                                    "username": "\$DOCKER_USERNAME",
                                    "password": "\$DOCKER_PASSWORD"
                                }
                            }
                        }
                        EOF
                        """
                        
                        // 3. Run Kaniko: Build image using Dockerfile and push immediately to both tags
                        sh "/kaniko/executor --context=\$(pwd) --dockerfile=\$(pwd)/Dockerfile --destination=${DOCKER_IMAGE} --destination=${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }
        
        // This stage reverts back to the 'agent any' pod (or uses the existing one)
        stage('Deploy to Kubernetes') {
            agent any
            steps {
                // Ensure the deployment file is updated with the new image tag
                sh "sed -i 's|bolatunj/devops-petcliinc:latest|${DOCKER_IMAGE}|g' k8s/deployment.yml"
                sh "kubectl apply -f k8s/service.yml"
                sh "kubectl apply -f k8s/deployment.yml"
                sh "kubectl rollout status deployment/${K8S_DEPLOYMENT_NAME} --namespace default"
            }
        }
    }
}
