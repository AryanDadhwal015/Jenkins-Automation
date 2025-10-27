import groovy.json.JsonOutput

pipeline {
  agent any

  environment {
    GIT_REPO_URL        = 'https://github.com/AryanDadhwal015/Jenkins-Automation.git'
    IMAGE_BASE_NAME     = 'my-app'
    CONTAINER_BASE_NAME = 'my-app'
    HOST_PORT           = '80'
    CONTAINER_PORT      = '80'
    INSTANCE_IP         = '35.154.161.144' // <-- your EC2 public IP
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
          def branchName = env.BRANCH_NAME ?: env.CHANGE_BRANCH ?: 'local'
          def sanitizedBranch = branchName.replaceAll('[^a-zA-Z0-9_.-]', '-').toLowerCase()
          env.IMAGE_TAG = "${sanitizedBranch}-${env.BUILD_NUMBER}"

          echo "Building Docker image: ${IMAGE_BASE_NAME}:${IMAGE_TAG}"
          sh "docker build -t ${IMAGE_BASE_NAME}:${IMAGE_TAG} ."
        }
      }
    }

    stage('Deploy Container') {
      steps {
        script {
          def isPR = env.CHANGE_ID != null
          def containerName = isPR ? "${CONTAINER_BASE_NAME}-pr-${env.CHANGE_ID}" : CONTAINER_BASE_NAME
          def port = isPR ? (10000 + env.BUILD_NUMBER.toInteger()) : 80
          def url = "http://${INSTANCE_IP}:${port}"

          // Stop existing container (if any)
          def existing = sh(script: "docker ps -aq -f name=${containerName}", returnStdout: true).trim()
          if (existing) {
            sh "docker stop ${existing}"
            sh "docker rm ${existing}"
          }

          // Run new container
          echo "Starting container ${containerName} on port ${port}"
          sh """
            docker run -d \
              --name ${containerName} \
              -p ${port}:${CONTAINER_PORT} \
              ${IMAGE_BASE_NAME}:${IMAGE_TAG}
          """

          echo "✅ App deployed on ${url}"

          // If it's a PR, notify GitHub
          if (isPR) {
            withCredentials([string(credentialsId: 'GITHUB_PR_TOKEN', variable: 'TOKEN')]) {
              def repoOwner = 'AryanDadhwal015'
              def repoName = 'Jenkins-Automation'
              def commentBody = """
              🚀 **Preview Environment Ready!**
              - **URL:** ${url}
              - **Image:** ${IMAGE_BASE_NAME}:${IMAGE_TAG}
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
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Build completed successfully."
    }
    failure {
      echo "❌ Build failed."
    }
  }
}
