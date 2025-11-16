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
  # 1. Explicitly define JNLP and mount the volume so it can write the config file.
  - name: jnlp 
    image: jenkins/inbound-agent:3345.v03dee9b_f88fc-1 
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker # JNLP can now safely write here!
    - name: workspace-volume 
      mountPath: /home/jenkins/agent
  
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    # Ensure the container stays alive using the BusyBox shell.
    command:
    - "/busybox/sh"
    - "-c"
    - "sleep 9999999"
    args:
    - "--context=/workspace"
    - "--dockerfile=/workspace/Dockerfile"
    - "--verbosity=info"
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
                // This script block runs on the stable JNLP container
                script {
                    def dockerConfigPath = "/kaniko/.docker/config.json"
                    
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        
                        // Use sh to create the directory on the shared volume
                        sh 'mkdir -p /kaniko/.docker' 

                        // Use Groovy writeFile to create the config.json (cleaner than shell redirection)
                        echo "Creating Docker config in ${dockerConfigPath}"
                        def authString = "${env.DOCKER_USER}:${env.DOCKER_PASS}".getBytes('UTF-8').encodeBase64().toString()
                        def dockerConfigContent = '{"auths":{"https://index.docker.io/v1/":{"auth":"' + authString + '"}}}'

                        writeFile(
                            file: dockerConfigPath,
                            text: dockerConfigContent
                        )
                        
                        // Switch to the 'kaniko' container
                        container('kaniko') {}
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
              agent {
                kubernetes {
                    yaml """
                    apiVersion: v1
                    kind: Pod
                    spec:
                    serviceAccountName: jenkins
                    containers:
                        - name: kubectl
                          image: bitnami/kubectl:latest
                    """
                    defaultContainer 'kubectl'
                }
            }
            steps {
                container('kubectl') {
                    sh """
                    kubectl apply -f k8s/deployment.yml
                    kubectl apply -f k8s/service.yml
                    kubectl set image deployment/${K8S_DEPLOYMENT_NAME} petclinic=${DOCKER_IMAGE} -n default
                    kubectl rollout status deployment/${K8S_DEPLOYMENT_NAME} -n default --timeout=5m
                    """
                }
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
