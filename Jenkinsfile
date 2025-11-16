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
    # Ensure the container stays alive
    command:
    - /busybox/sh
    - -c
    - 'sleep 9999999'
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
    - name: workspace-volume
      mountPath: /home/jenkins/agent
  volumes:
  - name: docker-config
    emptyDir: {}
"""
                    defaultContainer 'kaniko'
                    inheritFrom ''
                }
            }
            steps {
                container('kaniko') {
                    script {
                        withCredentials([usernamePassword(
                            credentialsId: 'dockerhub-credentials',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )]) {
                            // 👇 FIX: Use a secure, whitelisted way to encode Base64.
                            // We use Java/Groovy code that should be whitelisted (or requires a quick whitelisting) 
                            // as it is the most reliable way to get the string, and is generally preferred 
                            // over shell gymnastics.
                            def authString = "echo -n ${env.DOCKER_USER}:${env.DOCKER_PASS}".execute().text.encodeBase64().toString()
                            def dockerConfig = '{"auths":{"https://index.docker.io/v1/":{"auth":"' + authString + '"}}}'

                            sh """
                                # 1. Write the pre-encoded Docker config.json directly to Kaniko's expected path
                                printf '%s' '${dockerConfig}' > /kaniko/.docker/config.json
                                
                                # 2. Execute Kaniko
                                /kaniko/executor \\
                                  --context=/home/jenkins/agent/workspace/spring-petclinic-pipeline \\
                                  --dockerfile=Dockerfile \\
                                  --destination=${DOCKER_IMAGE} \\
                                  --destination=${DOCKER_HUB_USER}/${IMAGE_NAME}:latest
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
