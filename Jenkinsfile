pipeline {
  agent any

  environment {
    GIT_REPO_URL        = 'https://github.com/AryanDadhwal015/Jenkins-Automation.git'
    BRANCH_NAME         = 'main'
    IMAGE_BASE_NAME     = 'my-app'
    CONTAINER_BASE_NAME = 'my-app'
    HOST_PORT           = '80'
    CONTAINER_PORT      = '80'
    INSTANCE_IP         = '35.154.161.144' // <-- Change it to your IP address 
  }

  stages {
    stage('Clone from GitHub') {
      steps {
        echo "Cloning ${GIT_REPO_URL} (branch: ${BRANCH_NAME}) ..."
        checkout([$class: 'GitSCM',
          branches: [[name: "${BRANCH_NAME}"]],
          userRemoteConfigs: [[url: "${GIT_REPO_URL}"]]
        ])
      }
    }

    stage('Install Docker (if not present)') {
      steps {
        script {
          sh '''
            if ! which docker > /dev/null 2>&1; then
              sudo apt-get update -y
              sudo apt-get install -y ca-certificates curl gnupg lsb-release

              sudo mkdir -p /etc/apt/keyrings
              sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
              sudo chmod a+r /etc/apt/keyrings/docker.asc

              echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
              
              sudo apt-get update -y
              sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

              sudo usermod -aG docker $(whoami)
              sudo systemctl enable docker
              sudo systemctl start docker
            fi
            docker --version
          '''
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          env.IMAGE_TAG = "${env.BUILD_NUMBER}"
          echo "Building Docker image: ${IMAGE_BASE_NAME}:${IMAGE_TAG}"
          sh "docker build -t ${IMAGE_BASE_NAME}:${IMAGE_TAG} ."
        }
      }
    }

    stage('Deploy Container') {
      steps {
        script {
          def previousContainer = sh(
            script: "docker ps -aq -f name=${CONTAINER_BASE_NAME}",
            returnStdout: true
          ).trim()

          if (previousContainer) {
            echo "Stopping previous container: ${previousContainer}"
            sh "docker stop ${previousContainer}"
            sh "docker rm ${previousContainer}"
          }

          echo "Starting new container with image ${IMAGE_BASE_NAME}:${IMAGE_TAG}"
          sh """
            docker run -d \
              --name ${CONTAINER_BASE_NAME} \
              -p ${HOST_PORT}:${CONTAINER_PORT} \
              ${IMAGE_BASE_NAME}:${IMAGE_TAG}
          """
          echo "✅ Container started successfully and mapped to http://${INSTANCE_IP}:${HOST_PORT}"
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployment successful!"
      sh "docker ps"
    }
    failure {
      echo "❌ Deployment failed. Check logs above."
    }
  }
}
