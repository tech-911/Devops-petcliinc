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
        GIT_REPO = "https://github.com/tech-911/Devops-petcliinc.git"
    }

    stages {

        stage('Checkout & Unit Build (optional local build)') {
            steps {
                // Optional: local compile / tests on Jenkins agent (keeps your quick feedback).
                // Kaniko will build from the git repo directly, so this stage is informational/testing.
                git branch: 'main',
                    credentialsId: 'github-credentials',
                    url: "${env.GIT_REPO}"

                sh 'mvn clean package -DskipTests' // quick check; remove -DskipTests if you want tests
            }
        }

        stage('Build & Push with Kaniko (builds directly from git)') {
            /*
             * Key points:
             *  - Kaniko runs as its own container and uses its own entrypoint (/kaniko/executor)
             *  - We pass a git:// context so Kaniko clones the repo itself and avoids workspace race.
             *  - Docker auth is provided via a Kubernetes Secret mounted at /kaniko/.docker
             *  - You MUST create the secret `docker-config-secret` in the jenkins namespace beforehand.
             */
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

    # jnlp (jenkins agent) - used by Jenkins to run steps when necessary
    - name: jnlp
      image: jenkins/inbound-agent:latest
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
        - name: workspace-volume
          mountPath: /workspace

    # kaniko container - uses the executor entrypoint directly (no shell)
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      command:
        - /kaniko/executor
      args:
        - "--context=git://${GIT_REPO}"
        - "--dockerfile=/workspace/Dockerfile"
        - "--destination=${DOCKER_IMAGE}"
        - "--destination=${DOCKER_HUB_USER}/${IMAGE_NAME}:latest"
        - "--verbosity=info"
      # mount the docker config secret so Kaniko can authenticate
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
        - name: workspace-volume
          mountPath: /workspace

  volumes:
    - name: docker-config
      secret:
        secretName: docker-config-secret
    - name: workspace-volume
      emptyDir: {}
"""
                    defaultContainer 'jnlp'
                }
            }

            steps {
                script {
                    // Optional: show image name we will produce
                    echo "Will build and push image: ${DOCKER_IMAGE}"

                    // Note: Because we pass a git:// context, Kaniko clones the repo itself.
                    // We do not try to run /kaniko/executor via 'sh' (that fails because Kaniko has no shell).
                    // The kaniko container will be started by Kubernetes with the specified command/args,
                    // so it runs automatically and will push the image when it finishes.
                    //
                    // Jenkins will wait for the pod to complete and stream Kaniko logs to the job console.

                    echo "Started Kaniko container with git context. Waiting for it to complete..."
                }
            }
        }

        stage('Deploy to Kubernetes') {
            /*
             * This deploy stage runs in a small pod that has kubectl.
             * We use bitnami/kubectl with command 'cat' so the container stays alive for Jenkins to exec into.
             */
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
      command:
        - cat
      tty: true
"""
                    defaultContainer 'kubectl'
                }
            }

            steps {
                container('kubectl') {
                    // Apply resources and update the image, then wait for rollout
                    sh """
                      echo "Applying Kubernetes manifests..."
                      kubectl apply -f k8s/ns-prod.yaml
                      kubectl apply -f k8s/db.yml
                      kubectl apply -f k8s/petclinic.yml

                      echo "Updating deployment image..."
                      kubectl set image deployment/${K8S_DEPLOYMENT_NAME} \
                        petclinic=${DOCKER_IMAGE} -n default

                      echo "Waiting for rollout..."
                      kubectl rollout status deployment/${K8S_DEPLOYMENT_NAME} -n default --timeout=5m
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
