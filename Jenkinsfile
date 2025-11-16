pipeline {
  agent any

  tools {
    jdk 'JDK-17'
    maven 'M3'
  }

  environment {
    DOCKER_HUB_USER = 'bolatunj'
    IMAGE_NAME      = "devops-petcliinc"
    IMAGE_TAG       = "${env.BUILD_NUMBER}"
    DOCKER_IMAGE    = "${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
    K8S_DEPLOYMENT_NAME = "petclinic-deployment"
  }

  stages {

    /* ================================
       CHECKOUT CODE
       ================================ */
    stage('Checkout') {
      steps {
        git branch: 'main',
            credentialsId: 'github-credentials',
            url: 'https://github.com/tech-911/Devops-petcliinc.git'
      }
    }

    /* ================================
       MAVEN BUILD
       ================================ */
    stage('Build & Test') {
      steps {
        sh 'mvn -B clean package -DskipTests=false'
      }
      post {
        success { archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true }
      }
    }

    /* ================================
       FIXED KANIKO STAGE
       ================================ */
    stage('Build & Push Image (Kaniko)') {
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
    command:
      - /busybox/sleep
      - "999999"
    tty: true
    volumeMounts:
      - name: docker-config
        mountPath: /kaniko/.docker
      - name: workspace
        mountPath: /workspace
  volumes:
    - name: docker-config
      emptyDir: {}
    - name: workspace
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

              sh """
                echo "🔐 Creating Kaniko Docker credential..."
                mkdir -p /kaniko/.docker

cat <<EOF > /kaniko/.docker/config.json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "$(echo -n "${DOCKER_USER}:${DOCKER_PASS}" | base64)"
    }
  }
}
EOF

                echo "📦 Running Kaniko build..."
                /kaniko/executor \
                  --context=/workspace \
                  --dockerfile=/workspace/Dockerfile \
                  --destination=${DOCKER_IMAGE} \
                  --destination=${DOCKER_HUB_USER}/${IMAGE_NAME}:latest \
                  --single-snapshot
              """
            }
          }
        }
      }
    }

    /* ================================
       KUBERNETES DEPLOYMENT
       ================================ */
    stage('Deploy to Kubernetes') {
      agent any
      steps {
        sh """
          kubectl apply -f k8s/deployment.yml
          kubectl apply -f k8s/service.yml
          kubectl set image deployment/${K8S_DEPLOYMENT_NAME} petclinic=${DOCKER_IMAGE} -n default || true
          kubectl rollout status deployment/${K8S_DEPLOYMENT_NAME} -n default --timeout=5m || true
        """
      }
    }
  }

  post {
    success { echo '✅ Pipeline completed successfully!' }
    failure { echo '❌ Pipeline failed!' }
  }
}
