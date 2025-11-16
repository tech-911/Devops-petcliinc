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
        sh 'mvn -B clean package -DskipTests=false'
      }
      post {
        success { archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true }
      }
    }

    stage('Build & Push Image with Kaniko') {
      agent {
        kubernetes {
          yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    args: ["--skip-unused-stages"]
    volumeMounts:
      - name: docker-config
        mountPath: /kaniko/.docker
      - name: workspace
        mountPath: /workspace
  volumes:
    - name: docker-config
      secret:
        secretName: regcred
        items:
        - key: .dockerconfigjson
          path: config.json
    - name: workspace
      emptyDir: {}
"""
          defaultContainer 'kaniko'
        }
      }

      steps {
        container('kaniko') {
          sh """
            cp -r ${WORKSPACE}/* /workspace/

            /kaniko/executor \
              --context=/workspace \
              --dockerfile=/workspace/Dockerfile \
              --destination=${DOCKER_IMAGE} \
              --destination=${DOCKER_HUB_USER}/${IMAGE_NAME}:latest
          """
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh """
          kubectl apply -f k8s/deployment.yml
          kubectl apply -f k8s/service.yml
        """
      }
    }

  }

  post {
    success { echo 'Pipeline finished successfully.' }
    failure { echo 'Pipeline failed!' }
  }
}
