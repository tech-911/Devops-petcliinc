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
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build & Push with Kaniko') {
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      command:
        - /busybox/sh
      args:
        - -c
        - sleep 1000000
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
        - name: workspace-volume
          mountPath: /workspace

    - name: jnlp
      image: jenkins/inbound-agent:latest
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
        - name: workspace-volume
          mountPath: /workspace

  volumes:
    - name: docker-config
      emptyDir: {}
    - name: workspace-volume
      emptyDir: {}
"""
                    defaultContainer 'jnlp'
                }
            }

            steps {
                script {
                    def cfg = "/kaniko/.docker/config.json"

                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'USER',
                        passwordVariable: 'PASS'
                    )]) {

                        // Create auth file
                        sh "mkdir -p /kaniko/.docker"
                        def token = "${USER}:${PASS}".bytes.encodeBase64().toString()
                        writeFile file: cfg, text: """
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "${token}"
    }
  }
}
"""

                        echo "🔥 Running Kaniko build & push..."

                        container('kaniko') {
                            sh """
/kaniko/executor \
  --context=/workspace \
  --dockerfile=/workspace/Dockerfile \
  --destination=${DOCKER_IMAGE} \
  --verbosity=info
"""
                        }
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
      command: ['cat']
      tty: true
"""
                    defaultContainer 'kubectl'
                }
            }

            steps {
                container('kubectl') {
                    sh """
                        echo "Deploying..."

                        kubectl apply -f k8s/ns-prod.yaml
                        kubectl apply -f k8s/db.yml
                        kubectl apply -f k8s/petclinic.yml

                        kubectl set image deployment/${K8S_DEPLOYMENT_NAME} \
                            petclinic=${DOCKER_IMAGE} -n default

                        kubectl rollout status deployment/${K8S_DEPLOYMENT_NAME} \
                            -n default --timeout=5m
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
            echo 'Pipeline failed!'
        }
    }
}
