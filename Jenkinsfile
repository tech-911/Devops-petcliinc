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
  # IMPORTANT: Add nodeSelector to force scheduling to the PV node
  nodeSelector:
    kubernetes.io/hostname: k8s-worker1 
  serviceAccountName: jenkins
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    # FIX: Use /busybox/sh and sleep to keep the container alive for Jenkins
    command:
    - /busybox/sh
    - -c
    - 'sleep 9999999'
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
    # If using local-storage PV, you need to mount it here.
    # If the workspace-volume is EmptyDir (as it is below), this is okay.
    # If you were using kaniko-pvc, you'd also need the workspace-volume mount:
    # - name: workspace-volume
    #   mountPath: /home/jenkins/agent
  volumes:
  - name: docker-config
    emptyDir: {}
  # If you were using kaniko-pvc, you'd add it here:
  # - name: workspace-volume
  #   persistentVolumeClaim:
  #     claimName: kaniko-pvc
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
                            sh '''
                                echo "{\\"auths\\":{\\"https://index.docker.io/v1/\\":{\\"auth\\":\\"$(echo -n $DOCKER_USER:$DOCKER_PASS | base64)\\"}}}" > /kaniko/.docker/config.json
                                /kaniko/executor \\
                                  --context=/home/jenkins/agent/workspace/spring-petclinic-pipeline \\
                                  --dockerfile=Dockerfile \\
                                  --destination=''' + env.DOCKER_IMAGE + ''' \\
                                  --destination=''' + env.DOCKER_HUB_USER + '/' + env.IMAGE_NAME + ''':latest
                            '''
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
