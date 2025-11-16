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
  # CRITICAL FIX: Define JNLP container explicitly to give it the volume mount
  - name: jnlp 
    image: jenkins/inbound-agent:3345.v03dee9b_f88fc-1 # Standard JNLP image
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker # JNLP can now safely write here!
    - name: workspace-volume 
      mountPath: /home/jenkins/agent # Default workspace
  
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - "/busybox/sh"
    - "-c"
    - "sleep 9999999"
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
    - name: workspace-volume 
      mountPath: /home/jenkins/agent
  
  volumes:
  - name: docker-config
    emptyDir: {}
  - name: workspace-volume
    emptyDir: {}
"""
                    inheritFrom ''
                }
            }
            steps {
                // This script block runs on the JNLP container (default agent)
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        
                        // This mkdir command will now operate on the mounted emptyDir volume
                        sh 'mkdir -p /kaniko/.docker' 
                        
                        echo "Creating Docker config in /kaniko/.docker/config.json"
                        
                        // Create the config.json file with credentials
                        sh '''
                            set -e
                            
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
                            echo "Config created."
                        '''

                        // Switch to the 'kaniko' container to run the executor.
                        container('kaniko') {
                            sh """
                                echo "Starting Kaniko build in kaniko container..."
                                /kaniko/executor \\
                                  --context=${WORKSPACE} \\
                                  --dockerfile=${WORKSPACE}/Dockerfile \\
                                  --destination=${DOCKER_IMAGE} \\
                                  --destination=${DOCKER_HUB_USER}/${IMAGE_NAME}:latest \\
                                  --verbosity=info
                                echo "Kaniko build complete."
                            """
                        }
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
