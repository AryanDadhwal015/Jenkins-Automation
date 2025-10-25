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
        // Sanitize the branch name for Docker compatibility
        def branchName = env.BRANCH_NAME ?: 'local'
        def sanitizedBranch = branchName.replaceAll('[^a-zA-Z0-9_.-]', '-').toLowerCase()
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
        // Sanitize branch name again for Docker tag safety
        def branchName = env.BRANCH_NAME ?: 'local'
        def sanitizedBranch = branchName.replaceAll('[^a-zA-Z0-9_.-]', '-').replaceAll('/', '-').toLowerCase()
        def imageTag = "${sanitizedBranch}-${env.BUILD_NUMBER ?: '0'}"

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
        sh """
          docker run -d \
            --name ${env.CONTAINER_BASE_NAME} \
            -p 80:80 \
            ${env.IMAGE_BASE_NAME}:${imageTag}
        """

        echo "Container deployed successfully! Your application is serving on http://${env.SERVER_IP}:80"
    }
  }
}

