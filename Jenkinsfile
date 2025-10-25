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
          def sanitizedBranch = branchName
                                  .replaceAll('[^a-zA-Z0-9_.-]', '-')
                                  .replaceAll('/', '-')
                                  .toLowerCase()
          def imageTag = "${sanitizedBranch}-${env.BUILD_NUMBER ?: '0'}"

          echo "Building Docker image: ${env.IMAGE_BASE_NAME}:${imageTag}"
          sh "docker build -t ${env.IMAGE_BASE_NAME}:${imageTag} ."

          // Save tag for later stage
          env.IMAGE_TAG = imageTag
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

   stage('Post or Print URL') {
      steps {
        script {
          def previewUrl = "http://${env.SERVER_IP}:80"

          if (env.CHANGE_ID) {
            // 🟢 Case 1: Running for a PR — post comment on GitHub
            def prNumber = env.CHANGE_ID
            def repoUrl = env.GIT_URL
            def apiUrl = repoUrl
                .replace('https://github.com/', 'https://api.github.com/repos/')
                .replace('.git', '') + "/issues/${prNumber}/comments"

            withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
              sh """
                curl -X POST \
                  -H "Authorization: token ${GITHUB_TOKEN}" \
                  -H "Content-Type: application/json" \
                  -d '{"body": "${previewUrl}"}' \
                  ${apiUrl}
              """
            }

            echo "Posted deployment URL to GitHub PR #${prNumber}: ${previewUrl}"
          } else {
            // 🟡 Case 2: Normal/manual run — just print the URL in console
            echo "Application deployed successfully! Access it at: ${previewUrl}"
          }
        }
      }
    }

  } // end stages
} // end pipeline
