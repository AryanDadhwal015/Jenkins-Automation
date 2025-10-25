pipeline {
  agent any

  parameters {
    string(name: 'BRANCH_NAME', defaultValue: 'feat/deploy-test', description: 'Git branch to build')
  }

  environment {
    // Ensure a sensible default so preview/dev SCP deploys run when DEPLOY_METHOD isn't set in Jenkins
    DEPLOY_METHOD = "${env.DEPLOY_METHOD ?: ''}"
    BUILD_DIR = 'dist'
    // Image tag uses branch name and build number when available
    IMAGE_TAG = "${env.BRANCH_NAME ?: 'local'}-${env.BUILD_NUMBER ?: '0'}"
    IMAGE_BASE_NAME     = 'my-app'
    CONTAINER_BASE_NAME = 'my-app'
    SERVER_IP = '3.110.161.82' // <-- Change the IP with the actual IP of your Server
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    // // Quick debug to confirm branch and deploy method used by the pipeline
    // stage('Print Config') {
    //   steps {
    //     withCredentials([
    //       string(credentialsId: 'preview-path', variable: 'PREVIEW_PATH'),
    //       string(credentialsId: 'dev-deploy-path', variable: 'DEV_DEPLOY_PATH'),
    //       string(credentialsId: 'preview-server', variable: 'PREVIEW_SERVER')
    //     ]) {
    //       echo "Branch: ${env.BRANCH_NAME ?: 'local'}"
    //       echo "DEPLOY_METHOD: ${env.DEPLOY_METHOD}"
    //       echo "PREVIEW_PATH (from credentials): ${PREVIEW_PATH}"
    //       echo "DEV_DEPLOY_PATH (from credentials): ${DEV_DEPLOY_PATH}"
    //       echo "PREVIEW_SERVER (from credentials): ${PREVIEW_SERVER}"
    //     }
    //   }
    // }

     stage('Build Docker Image') {
      steps {
        script {
          // Sanitize branch name for Docker tag
          def sanitizedBranch = (env.BRANCH_NAME ?: 'local')
                                  .replaceAll('[^a-zA-Z0-9_.-]', '-') // Replace invalid chars
                                  .toLowerCase()
    
          env.IMAGE_TAG = "${sanitizedBranch}-${env.BUILD_NUMBER ?: '0'}"
    
          echo "Building Docker image: ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG}"
          sh "docker build -t ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG} ."
    }
  }
}


    stage('Deploy Container') {
      steps {
        script {
          def previousContainer = sh(
            script: "docker ps -aq -f name=${env.CONTAINER_BASE_NAME}",
            returnStdout: true
          ).trim()

          if (previousContainer) {
            echo "Stopping previous container: ${previousContainer}"
            sh "docker stop ${previousContainer}"
            sh "docker rm ${previousContainer}"
          }

          echo "Starting new container with image ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG}"
          sh """
            docker run -d \
              --name ${env.CONTAINER_BASE_NAME} \
              -p 80:80 \
              ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG}
          """

          echo "Container deployed successfully! Your application is serving on http://${env.SERVER_IP}:80"
        }
      }
    }
  }
}
