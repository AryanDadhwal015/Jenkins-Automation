pipeline {
  agent { label 'slave-1' }

  environment {
    GIT_REPO_URL        = 'https://github.com/AryanDadhwal015/Jenkins-Automation.git'
    BRANCH_NAME         = 'main'
    IMAGE_BASE_NAME     = 'my-app'
    CONTAINER_BASE_NAME = 'my-app'
  }

  stages {
    stage('Clone from GitHub') {
      steps {
        echo "Cloning ${GIT_REPO_URL} (branch: ${BRANCH_NAME}) ..."
        checkout([$class: 'GitSCM',
          branches: [[name: "${BRANCH_NAME}"]],
          userRemoteConfigs: [[
            url: "${GIT_REPO_URL}"
          ]]
        ])
      }
    }

    stage('Install Docker (if not present)') {
      steps {
        script {
          sh '''
            if ! command -v docker &> /dev/null; then
              sudo apt-get update -y
              sudo apt-get install -y ca-certificates curl gnupg lsb-release
              sudo mkdir -p /etc/apt/keyrings
              curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o docker.gpg
              sudo gpg --dearmor --batch --yes -o /etc/apt/keyrings/docker.gpg docker.gpg
              rm docker.gpg
              echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | sudo tee /etc/apt/sources.list.d/docker.list
              sudo apt-get update -y
              sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
              sudo usermod -aG docker ubuntu
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
            script: "docker ps -q -f name=${CONTAINER_BASE_NAME}",
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
              -p 80:80 \
              ${IMAGE_BASE_NAME}:${IMAGE_TAG}
          """
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
