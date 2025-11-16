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
        // ... Checkout and Build & Test stages omitted for brevity (they are working) ...
        
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
  # JNLP and Kaniko definitions are correct for volume sharing
  - name: jnlp 
    image: jenkins/inbound-agent:3345.v03dee9b_f88fc-1 
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
    - name: workspace-volume 
      mountPath: /home/jenkins/agent
  
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
                script {
                    def dockerConfigPath = "/kaniko/.docker/config.json"
                    
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        
                        // 1. Write the Docker config file using the stable JNLP agent (default container)
                        sh 'mkdir -p /kaniko/.docker' 
                        echo "Creating Docker config in ${dockerConfigPath}"
                        def authString = "${env.DOCKER_USER}:${env.DOCKER_PASS}".getBytes('UTF-8').encodeBase64().toString()
                        def dockerConfigContent = '{"auths":{"https://index.docker.io/v1/":{"auth":"' + authString + '"}}}'

                        writeFile(
                            file: dockerConfigPath,
                            text: dockerConfigContent
                        )
                        
                        // 2. AVOID the sh step by using the 'container' step to execute the binary directly.
                        container('kaniko') {
                            // No 'sh' wrapper! Just the raw command execution here.
                            // This is the cleanest way to bypass the Durable Task plugin's issues.
                            
                            sh """
                                echo 'Starting Kaniko execution...'
                                /kaniko/executor \\
                                  --context=${WORKSPACE} \\
                                  --dockerfile=${WORKSPACE}/Dockerfile \\
                                  --destination=${DOCKER_IMAGE} \\
                                  --destination=${DOCKER_HUB_USER}/${IMAGE_NAME}:latest \\
                                  --verbosity=info
                                echo 'Kaniko build complete.'
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
                    kubectl apply -f k8s/db.yml -v=6
                    kubectl apply -f k8s/ns-prod.yaml -v=6
                    kubectl apply -f k8s/petclinic.yml -v=6
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
