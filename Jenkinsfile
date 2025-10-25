pipeline {
  agent any

  parameters {
    string(name: 'BRANCH_NAME', defaultValue: 'feat/deploy-test', description: 'Git branch to build')
  }

  environment {
    GIT_REPO_URL        = 'https://github.com/AryanDadhwal015/Jenkins-Automation.git'
    IMAGE_BASE_NAME     = 'my-app'
    CONTAINER_BASE_NAME = 'my-app'
    HOST_PORT           = '80'
    CONTAINER_PORT      = '80'
    INSTANCE_IP         = '35.154.161.144' // <-- Change it to your IP address 
  }

  stages {
    stage('Clone from GitHub') {
      steps {
        echo "Cloning ${GIT_REPO_URL} (branch: ${params.BRANCH_NAME}) ..."
        checkout([$class: 'GitSCM',
          branches: [[name: "${params.BRANCH_NAME}"]],
          userRemoteConfigs: [[url: "${GIT_REPO_URL}"]]
        ])
      }
    }
        
  stage('Build Docker Image') {
        steps {
            script {
                  def sanitizedBranch = (env.BRANCH_NAME ?: 'local')
                                          .replaceAll('[^a-zA-Z0-9_.-]', '-')
                                          .toLowerCase()
                    env.IMAGE_TAG = "${sanitizedBranch}-${env.BUILD_NUMBER ?: '0'}"

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
