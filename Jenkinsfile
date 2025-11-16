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
    # FIX 1: Ensure the container stays alive using a known shell (BusyBox)
    command:
    - "/busybox/sh"
    - "-c"
    - "sleep 9999999"
    volumeMounts:
    # This volume mount path is shared by all containers in the Pod
    - name: docker-config
      mountPath: /kaniko/.docker
    # This mount allows Kaniko to see the source code
    - name: workspace-volume 
      mountPath: /home/jenkins/agent
  volumes:
    # Define the shared volume for the Docker config
  - name: docker-config
    emptyDir: {}
"""
                    inheritFrom ''
                }
            }
            steps {
                script {
                    // This entire block runs in the stable JNLP container (the default agent)
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        
                        // 1. Write the config file directly to the shared volume path.
                        // The JNLP agent is stable and uses 'writeFile' (Groovy), avoiding the shell issue.
                        echo "Writing Docker config to shared volume path: /kaniko/.docker/config.json"
                        def authString = "${env.DOCKER_USER}:${env.DOCKER_PASS}".getBytes('UTF-8').encodeBase64().toString()
                        def dockerConfigContent = '{"auths":{"https://index.docker.io/v1/":{"auth":"' + authString + '"}}}'

                        sh "mkdir -p /kaniko/.docker" // Ensure the folder exists on the shared volume
                        writeFile(
                            file: "/kaniko/.docker/config.json",
                            text: dockerConfigContent
                        )
                        
                        // 2. Switch to the minimal 'kaniko' container to run the executor.
                        // We are running the shortest possible command here to minimize the chance of the durable task failing.
                        container('kaniko') {
                            sh """
                                /kaniko/executor \\
                                  --context=${WORKSPACE} \\
                                  --dockerfile=${WORKSPACE}/Dockerfile \\
                                  --destination=${DOCKER_IMAGE} \\
                                  --destination=${DOCKER_HUB_USER}/${IMAGE_NAME}:latest \\
                                  --verbosity=info
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
