pipeline {
    agent any
    
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
            steps {
                git branch: 'main',
                    credentialsId: 'github-credentials',
                    url: 'https://github.com/tech-911/Devops-petcliinc.git'
            }
        }
        
        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Build & Push with Kaniko') {
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: agent
spec:
  serviceAccountName: jenkins
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    # CRITICAL FIX 1: Ensure the container uses a BusyBox shell to stay alive
    # (Fixes Kaniko exit code 1)
    command:
    - "/busybox/sh"
    - "-c"
    - "sleep 9999999"
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
    # CRITICAL FIX 2: Re-add the necessary workspace volume mount 
    # so Kaniko can access your source code and the agent can operate.
    - name: workspace-volume 
      mountPath: /home/jenkins/agent
  volumes:
  - name: docker-config
    emptyDir: {}
  # The workspace-volume definition is usually added automatically, 
  # but including it here is safer if you use an 'inheritFrom' template.
  - name: workspace-volume
    emptyDir: {}
"""
                    inheritFrom ''
                }
            }
            steps {
                // The container step is used here, which is causing the durable task issue.
                // We will try to resolve it by ensuring the shell is available.
                container('kaniko') {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        // The entire block runs inside the minimal Kaniko container shell.
                        // We use single quotes (''') for the sh block to prevent groovy interpolation 
                        // and ensure the shell syntax works cleanly.
                        sh '''
                            set -e
                            
                            # Log and ensure the correct workspace is used
                            echo "Workspace is: ${WORKSPACE}"
                            
                            echo "Creating Docker config in /kaniko/.docker/config.json"
                            
                            # Use mkdir -p to ensure the directory exists before writing the file
                            mkdir -p /kaniko/.docker
                            
                            # Create the config.json file with credentials
                            echo "Generating base64 auth string..."
                            AUTH_STRING=$(echo -n $DOCKER_USER:$DOCKER_PASS | base64)
                            
                            cat > /kaniko/.docker/config.json << EOFCONFIG
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "${AUTH_STRING}"
    }
  }
}
EOFCONFIG
                            echo "Config created. Starting Kaniko build..."
                            
                            /kaniko/executor \\
                              --context=${WORKSPACE} \\
                              --dockerfile=${WORKSPACE}/Dockerfile \\
                              --destination=${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} \\
                              --destination=${DOCKER_HUB_USER}/${IMAGE_NAME}:latest \\
                              --verbosity=info
                            
                            echo "Kaniko build complete."
                        '''
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            agent any
            steps {
                sh """
                    kubectl apply -f k8s/deployment.yml
                    kubectl apply -f k8s/service.yml
                    kubectl set image deployment/${K8S_DEPLOYMENT_NAME} petclinic=${DOCKER_IMAGE} -n default
                    kubectl rollout status deployment/${K8S_DEPLOYMENT_NAME} -n default --timeout=5m
                """
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
    }
}
