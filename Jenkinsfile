import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        BUILD_DIR = 'dist'
        IMAGE_BASE_NAME = 'my-app'
        CONTAINER_BASE_NAME = 'my-app'
        INSTANCE_IP = '15.206.80.224'
        GIT_REPO_URL = 'https://github.com/AryanDadhwal015/Jenkins-Automation.git'
        CONTAINER_PORT = '80'
        HOST_PORT = '80'
    }

    stages {
        stage('Check PR') {
            steps {
                script {
                    if (!env.CHANGE_ID) {
                        echo "This is not a PR. Skipping build."
                        currentBuild.result = 'SUCCESS'
                        return
                    }
                }
            }
        }

        stage('Clone from GitHub') {
            when { expression { env.CHANGE_ID != null } }
            steps {
                echo "Cloning ${GIT_REPO_URL} (PR branch: ${BRANCH_NAME}) ..."
                checkout([$class: 'GitSCM',
                    branches: [[name: "${BRANCH_NAME}"]],
                    userRemoteConfigs: [[url: "${GIT_REPO_URL}"]]
                ])
            }
        }

        stage('Build Docker Image') {
            when { expression { env.CHANGE_ID != null } }
            steps {
                script {
                    def sanitizedBranch = (env.CHANGE_BRANCH ?: 'pr').replaceAll('[^a-zA-Z0-9_.-]', '-').toLowerCase()
                    env.SANITIZED_BRANCH = sanitizedBranch
                    env.IMAGE_TAG = "${sanitizedBranch}-${env.BUILD_NUMBER}"

                    echo "Building Docker image: ${IMAGE_BASE_NAME}:${IMAGE_TAG}"
                    sh "docker build -t ${IMAGE_BASE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Deploy Container') {
            when { expression { env.CHANGE_ID != null } }
            steps {
                script {
                    def containerName = "${CONTAINER_BASE_NAME}-${env.SANITIZED_BRANCH}"
                    def url = "http://${INSTANCE_IP}:${HOST_PORT}"

                    def previousContainer = sh(script: "docker ps -aq -f name=${containerName}", returnStdout: true).trim()
                    if (previousContainer) {
                        echo "Stopping previous container: ${previousContainer}"
                        sh "docker stop ${previousContainer}"
                        sh "docker rm ${previousContainer}"
                    }

                    echo "Starting new container ${containerName} on port ${HOST_PORT}"
                    sh """
                        docker run -d \
                        --name ${containerName} \
                        -p ${HOST_PORT}:${CONTAINER_PORT} \
                        ${IMAGE_BASE_NAME}:${IMAGE_TAG}
                    """

                    echo "✅ Container started at ${url}"

                    // Post preview URL back to PR
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

    post {
        success { echo "✅ Deployment successful!" }
        failure { echo "❌ Deployment failed. Check logs above." }
    }
}
