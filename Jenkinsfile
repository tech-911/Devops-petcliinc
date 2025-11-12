pipeline {
    agent any // Back to running on the Jenkins controller node
    
    // Tools directive is back
    tools {
        jdk 'JDK-17' // Will install JDK 17
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
        // ... (Stages are the same as before, but now run on agent any)
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github-credentials', url: 'https://github.com/tech-911/Devops-petcliinc.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean install -DskipTests' 
                sh 'mvn test' 
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // This now executes on the host/controller node where the Docker CLI must be available (and linked to K8s)
                    docker.build(DOCKER_IMAGE)
                }
            }
        }
        // ... (Push and Deploy stages remain the same)
    }
}
