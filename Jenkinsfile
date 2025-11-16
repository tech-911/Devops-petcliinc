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

        /* ---------------------------------------------------------
           KANIKO BUILD STAGE — FULLY FIXED, WORKING VERSION
        --------------------------------------------------------- */
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

    - name: jnlp
      image: jenkins/inbound-agent:latest
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
        - name: workspace-volume
          mountPath: /workspace

    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      command: ["/bin/sh"]
      args: ["-c", "sleep infinity"]
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
                    def dockerConfigPath = "/kaniko/.docker/config.json"

                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {

                        sh 'mkdir -p /kaniko/.docker'

                        echo "Writing Docker Hub auth..."

                        def authString = "${env.DOCKER_USER}:${env.DOCKER_PASS}"
                                .getBytes("UTF-8")
                                .encodeBase64()
                                .toString()

                        def dockerConfig = """
                        {
                          "auths": {
                            "https://index.docker.io/v1/": {
                              "auth": "${authString}"
                            }
                          }
                        }
                        """

                        writeFile file: dockerConfigPath, text: dockerConfig.trim()

                        echo "🔥 Running Kaniko build & push..."

                        container('kaniko') {
                            sh """
                            /kaniko/executor \
                              --context=/workspace \
                              --dockerfile=/workspace/Dockerfile \
                              --destination=${DOCKER_IMAGE} \
                              --destination=${DOCKER_HUB_USER}/${IMAGE_NAME}:latest \
                              --verbosity=info
                            """
                        }
                    }
                }
            }
        }

        /* ---------------------------------------------------------
           KUBECTL STAGE — FULLY FIXED & WORKING
        --------------------------------------------------------- */
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
      command: ["/bin/sh"]
      args: ["-c", "sleep infinity"]
"""
                    defaultContainer 'kubectl'
                }
            }

            steps {
                container('kubectl') {
                    sh """
                    kubectl apply -f k8s/db.yml
                    kubectl apply -f k8s/ns-prod.yaml
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
            echo '✅ Pipeline completed SUCCESSFULLY!'
        }
        failure {
            echo '❌ Pipeline FAILED!'
        }
    }
}
