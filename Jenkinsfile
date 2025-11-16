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
    stage('Checkout') {
      steps {
        git branch: 'main',
            credentialsId: 'github-credentials',
            url: 'https://github.com/tech-911/Devops-petcliinc.git'
      }
    }

    stage('Build & Test') {
      steps {
        // Build the jar on a full agent (has Maven because of tools{})
        sh 'mvn -B clean package -DskipTests=false'
      }
      post {
        success { archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true }
      }
    }

    stage('Build & Push with Kaniko') {
      agent {
        kubernetes {
          // The YAML below is a minimal Kaniko pod. IMPORTANT: this **does not** inherit
          // Maven/Git tool config from the parent, so Jenkins won't try to install tools here.
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
    - /busybox/cat
    tty: true
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
          // ensure the Kaniko step uses the kaniko container by default and that the pod
          // does not try to inherit tools/installers from the parent node
          defaultContainer 'kaniko'
          inheritFrom ''
        }
      }

      steps {
        // copy the built artifact (target/*.jar) into the Kaniko context if Dockerfile expects it
        // Using ${WORKSPACE} so this uses the same workspace folder Jenkins already populated.
        container('kaniko') {
          script {
            withCredentials([usernamePassword(
              credentialsId: 'dockerhub-credentials',
              usernameVariable: 'DOCKER_USER',
              passwordVariable: 'DOCKER_PASS'
            )]) {
              sh """
                # prepare docker auth for kaniko
                mkdir -p /kaniko/.docker
                echo '{ "auths": { "https://index.docker.io/v1/": { "auth": "${DOCKER_USER}:${DOCKER_PASS}" | base64 -w0 } } }' > /kaniko/.docker/config.json || true

                # Kaniko expects a context directory. We mount workspace to /workspace,
                # so point context there. Ensure Dockerfile path is correct.
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
