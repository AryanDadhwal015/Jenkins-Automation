pipeline {
  agent any

  parameters {
    string(name: 'BRANCH_NAME', defaultValue: 'feat/deploy-test', description: 'Git branch to build')
  }

  environment {
    DEPLOY_METHOD = "${env.DEPLOY_METHOD ?: ''}"
    BUILD_DIR = 'dist'
    IMAGE_BASE_NAME     = 'my-app'
    CONTAINER_BASE_NAME = 'my-app'
    SERVER_IP = '13.201.117.191'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          // Sanitize branch name for Docker image tag
          def branchName = env.BRANCH_NAME ?: 'local'
          def sanitizedBranch = branchName.replaceAll('[^a-zA-Z0-9_.-]', '-').replaceAll('/', '-').toLowerCase()
          def imageTag = "${sanitizedBranch}-${env.BUILD_NUMBER ?: '0'}"

          echo "Building Docker image: ${env.IMAGE_BASE_NAME}:${imageTag}"
          sh "docker build -t ${env.IMAGE_BASE_NAME}:${imageTag} ."

          // Update environment variable for later stages
          env.IMAGE_TAG = imageTag
        }
      }
    }

    stage('Deploy Container') {
      steps {
        script {
          // Always use sanitized image tag
          def imageTag = env.IMAGE_TAG

          def previousContainer = sh(
            script: "docker ps -aq -f name=${env.CONTAINER_BASE_NAME}",
            returnStdout: true
          ).trim()

          if (previousContainer) {
            echo "Stopping previous container: ${previousContainer}"
            sh "docker stop ${previousContainer}"
            sh "docker rm ${previousContainer}"
          }

          echo "Starting new container with image ${env.IMAGE_BASE_NAME}:${imageTag}"
          sh "docker run -d --name ${env.CONTAINER_BASE_NAME} -p 80:80 ${env.IMAGE_BASE_NAME}:${imageTag}"

          echo "Container deployed successfully! Your application is serving on http://${env.SERVER_IP}:80"
        }
      }
    }

  } // end stages
} // end pipeline
