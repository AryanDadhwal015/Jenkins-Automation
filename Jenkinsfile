pipeline {
  agent any

  environment {
    GITHUB_USER        = 'AryanDadhwal015'
    GITHUB_REPO_NAME   = 'Jenkins-Automation'
    INSTANCE_IP        = '3.110.216.133'
    CONTAINER_PORT     = '80'
    HOST_PORT          = '8002'   // 👈 PR previews will run here
  }

  stages {

    stage('Clone from GitHub') {
      when { expression { env.CHANGE_ID != null } }
      steps {
        echo "🌀 Cloning ${env.GITHUB_REPO_NAME} for PR #${env.CHANGE_ID}"
        checkout scm
      }
    }

    stage('Build Docker Image') {
      when { expression { env.CHANGE_ID != null } }
      steps {
        script {
          def branch = env.CHANGE_BRANCH ?: env.BRANCH_NAME
          def sanitizedBranch = branch.replaceAll('[^a-zA-Z0-9_.-]', '-').toLowerCase()
          def repoNameLower = env.GITHUB_REPO_NAME.toLowerCase()

          env.SANITIZED_BRANCH = sanitizedBranch
          env.IMAGE_NAME = "${repoNameLower}:${sanitizedBranch}-${env.BUILD_NUMBER}"

          echo "🐳 Building Docker image: ${env.IMAGE_NAME}"

          sh "docker build -t ${env.IMAGE_NAME} ."
        }
      }
    }

    stage('Deploy Container') {
      when { expression { env.CHANGE_ID != null } }
      steps {
        script {
          def repoNameLower = env.GITHUB_REPO_NAME.toLowerCase()
          def containerName = "${repoNameLower}-${env.SANITIZED_BRANCH}"

          echo "🚀 Deploying container: ${containerName}"

          // Stop old container if exists
          sh """
            old_id=\$(docker ps -aq -f name=${containerName})
            if [ ! -z "\$old_id" ]; then
              echo "Stopping existing container..."
              docker stop \$old_id || true
              docker rm \$old_id || true
            fi
          """

          // Run new container on port 8002
          sh """
            docker run -d --name ${containerName} -p ${HOST_PORT}:${CONTAINER_PORT} ${env.IMAGE_NAME}
          """

          env.PREVIEW_URL = "http://${INSTANCE_IP}:${HOST_PORT}"
          echo "🌐 Preview running at: ${env.PREVIEW_URL}"
        }
      }
    }

    stage('Post Preview Link to GitHub PR') {
      when { expression { env.CHANGE_ID != null } }
      steps {
        script {
          withCredentials([usernamePassword(
            credentialsId: 'github_login',
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_TOKEN'
          )]) {
            echo "💬 Posting preview link to PR #${env.CHANGE_ID}"
            sh """
              curl -s -u ${GIT_USER}:${GIT_TOKEN} \
                   -H "Accept: application/vnd.github.v3+json" \
                   -d '{"body": "🚀 Preview for PR #${env.CHANGE_ID} is live at: ${env.PREVIEW_URL}"}' \
                   https://api.github.com/repos/${GITHUB_USER}/${GITHUB_REPO_NAME}/issues/${env.CHANGE_ID}/comments
            """
          }
        }
      }
    }
  }

  post {
    failure {
      echo "❌ Build failed for PR #${env.CHANGE_ID}"
    }
    success {
      echo "✅ Build succeeded for PR #${env.CHANGE_ID}"
    }
  }
}
