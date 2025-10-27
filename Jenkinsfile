import groovy.json.JsonOutput

pipeline {
  agent any

  environment {
    GIT_REPO_URL        = 'https://github.com/AryanDadhwal015/Jenkins-Automation.git'
    BRANCH_NAME         = 'main'
    IMAGE_BASE_NAME     = 'my-app'
    CONTAINER_BASE_NAME = 'my-app'
    HOST_PORT           = '80'
    CONTAINER_PORT      = '80'
    INSTANCE_IP         = '172.31.76.29' // <-- Change it to your IP address 172.31.76.29:8080
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


    stage('Build Docker Image') {
      steps {
        script {
          // Sanitize branch name for Docker tag and container usage
          def branch = env.BRANCH_NAME ?: env.CHANGE_BRANCH ?: 'local'
          def sanitizedBranch = branch.replaceAll('[^a-zA-Z0-9_.-]', '-').toLowerCase()
          env.SANITIZED_BRANCH = sanitizedBranch
          env.IMAGE_TAG = "${sanitizedBranch}-${env.BUILD_NUMBER}"

          echo "Building Docker image: ${IMAGE_BASE_NAME}:${IMAGE_TAG}"
          sh "docker build -t ${IMAGE_BASE_NAME}:${IMAGE_TAG} ."
        }
      }
    }

    stage('Deploy Container') {
      steps {
        script {
          // Use sanitized branch in container name
          def containerName = "${CONTAINER_BASE_NAME}-${env.SANITIZED_BRANCH}"
          def url = "http://${INSTANCE_IP}:${HOST_PORT}"

          // Stop previous container if exists
          def previousContainer = sh(script: "docker ps -aq -f name=${containerName}", returnStdout: true).trim()
          if (previousContainer) {
            echo "Stopping previous container: ${previousContainer}"
            sh "docker stop ${previousContainer}"
            sh "docker rm ${previousContainer}"
          }

          // Run new container
          echo "Starting new container ${containerName}"
          sh """
            docker run -d \
              --name ${containerName} \
              -p ${HOST_PORT}:${CONTAINER_PORT} \
              ${IMAGE_BASE_NAME}:${IMAGE_TAG}
          """
          echo "✅ Container started at http:"//${INSTANCE_IP}:${HOST_PORT}"

          // Post PR comment if this is a pull request
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
          }
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
