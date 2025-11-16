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
    # CRITICAL FIX: This command keeps the Kaniko sidecar container alive
    # until the Jenkins agent is ready to run the build command inside it.
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
"""
                    inheritFrom '' // Ensures JNLP remains the default container for utility tasks
                }
            }
            steps {
                script {
                    def dockerConfigPath = "/home/jenkins/agent/workspace/spring-petclinic-pipeline/docker-config.json"

                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        def authString = "${env.DOCKER_USER}:${env.DOCKER_PASS}".getBytes('UTF-8').encodeBase64().toString()
                        def dockerConfigContent = '{"auths":{"https://index.docker.io/v1/":{"auth":"' + authString + '"}}}'

                        // 1. Write the config file using the stable JNLP container
                        writeFile(
                            file: dockerConfigPath,
                            text: dockerConfigContent
                        )
                        
                        // 2. Use the 'kaniko' container to run the executor command
                        container('kaniko') {
                            sh """
                                /kaniko/executor \\
                                  --context=/home/jenkins/agent/workspace/spring-petclinic-pipeline \\
                                  --dockerfile=Dockerfile \\
                                  --destination=${DOCKER_IMAGE} \\
                                  --destination=${DOCKER_HUB_USER}/${IMAGE_NAME}:latest \\
                                  --docker-config=${dockerConfigPath}  
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
}
