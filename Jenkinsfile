import groovy.json.JsonOutput
import java.security.MessageDigest

pipeline {
  agent any

  environment {
    IMAGE_BASE_NAME = 'my-app'
    CONTAINER_BASE_NAME = 'my-app'
    INSTANCE_IP = '13.126.74.186'
    GIT_REPO_URL = 'https://github.com/AryanDadhwal015/Jenkins-Automation.git'
    CONTAINER_PORT = '80'
  }

  stages {
    stage('Clone from GitHub') {
      steps {
        echo "📦 Cloning ${GIT_REPO_URL} (branch: ${BRANCH_NAME}) ..."
        checkout([$class: 'GitSCM',
          branches: [[name: "${BRANCH_NAME}"]],
          userRemoteConfigs: [[url: "${GIT_REPO_URL}"]]
        ])
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          def branch = env.BRANCH_NAME ?: env.CHANGE_BRANCH ?: 'local'
          def sanitizedBranch = branch.replaceAll('[^a-zA-Z0-9_.-]', '-').toLowerCase()
          env.SANITIZED_BRANCH = sanitizedBranch
          env.IMAGE_TAG = "${sanitizedBranch}-${env.BUILD_NUMBER}"

          echo "🐳 Building Docker image: ${IMAGE_BASE_NAME}:${IMAGE_TAG}"
          sh "docker build -t ${IMAGE_BASE_NAME}:${IMAGE_TAG} ."
        }
      }
    }

    stage('Deploy Container') {
      steps {
        script {
          def containerName = "${CONTAINER_BASE_NAME}-${env.SANITIZED_BRANCH}"

          // 🔢 Deterministic port selection
          def basePort = env.CHANGE_ID ? 10000 : 11000
          def uniqueKey = env.CHANGE_ID ?: env.BRANCH_NAME
          def hash = MessageDigest.getInstance("MD5")
                                .digest(uniqueKey.bytes)
                                .collect { String.format("%02x", it) }
                                .join()
          // Convert hash → integer offset 1–900
          def offset = (Math.abs(hash.hashCode()) % 900) + 1
          def HOST_PORT = basePort + offset

          def url = "http://${INSTANCE_IP}:${HOST_PORT}"
          echo "🧩 Using deterministic port: ${HOST_PORT} for ${env.CHANGE_ID ? 'PR #' + env.CHANGE_ID : env.BRANCH_NAME}"

          // Stop & remove old container if it exists
          def existing = sh(script: "docker ps -aq -f name=${containerName}", returnStdout: true).trim()
          if (existing) {
            echo "🛑 Stopping previous container: ${existing}"
            sh "docker stop ${existing} && docker rm ${existing}"
          }

          // Run the new container
          echo "🚀 Starting new container ${containerName}..."
          sh """
            docker run -d \
              --name ${containerName} \
              -p ${HOST_PORT}:${CONTAINER_PORT} \
              ${IMAGE_BASE_NAME}:${IMAGE_TAG}
          """

          echo "✅ Container started at ${url}"

          // Comment on PR if it's a preview
          if (env.CHANGE_ID) {
            withCredentials([string(credentialsId: 'GITHUB_PR_TOKEN', variable: 'TOKEN')]) {
              def repoOwner = 'AryanDadhwal015'
              def repoName = 'Jenkins-Automation'
              def commentBody = """
              🚀 **Preview Environment Ready!**
              - **URL:** ${url}
              - **Image:** ${IMAGE_BASE_NAME}:${IMAGE_TAG}
              - **Branch:** ${env.SANITIZED_BRANCH}
              """

              def jsonPayload = JsonOutput.toJson([body: commentBody])

              sh """
                curl -s -X POST \
                  -H "Authorization: token \${TOKEN}" \
                  -H "Content-Type: application/json" \
                  -d '${jsonPayload}' \
                  https://api.github.com/repos/${repoOwner}/${repoName}/issues/${env.CHANGE_ID}/comments
              """
            }
          } else {
            echo "🌐 Merge build deployed at ${url}"
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployment successful!"
      sh "docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'"
    }
    failure {
      echo "❌ Deployment failed. Check lo
