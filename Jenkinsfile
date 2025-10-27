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
        PR_PREVIEW_PORT = 8085
    }

    stages {
        stage('Check Build Type') {
            steps {
                script {
                    echo "Branch name: ${env.BRANCH_NAME}"
                    echo "PR ID: ${env.CHANGE_ID ?: 'N/A'}"
                    echo "Target branch: ${env.CHANGE_TARGET ?: 'N/A'}"
                }
            }
        }

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
                    def sanitizedBranch = (env.CHANGE_BRANCH ?: env.BRANCH_NAME ?: 'branch')
                        .replaceAll('[^a-zA-Z0-9_.-]', '-')
                        .toLowerCase()

                    env.SANITIZED_BRANCH = sanitizedBranch
                    env.IMAGE_TAG = "${sanitizedBranch}-${env.BUILD_NUMBER}"

                    echo "Building Docker image: ${IMAGE_BASE_NAME}:${IMAGE_TAG}"
                    sh "docker build -t ${IMAGE_BASE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        // --- PR Preview Deployment ---
        stage('Deploy PR Preview') {
            when { expression { env.CHANGE_ID != null } }  // runs for any PR
            steps {
                script {
                    def containerName = "${CONTAINER_BASE_NAME}-pr"
                    def url = "http://${INSTANCE_IP}:${PR_PREVIEW_PORT}"

                    echo "Deploying PR #${env.CHANGE_ID} preview at ${url}"

                    // Stop previous container if exists
                    def prevContainer = sh(script: "docker ps -aq -f name=${containerName}", returnStdout: true).trim()
                    if (prevContainer) {
                        sh "docker stop ${prevContainer}"
                        sh "docker rm ${prevContainer}"
                    }

                    // Run new preview container on port 8085
                    sh """
                        docker run -d \
                        --name ${containerName} \
                        -p ${PR_PREVIEW_PORT}:${CONTAINER_PORT} \
                        ${IMAGE_BASE_NAME}:${IMAGE_TAG}
                    """

                    // Post URL back to PR
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

        // --- Main branch deployment ---
        stage('Deploy Main Branch') {
            when { branch 'main' }
            steps {
                script {
                    def containerName = "${CONTAINER_BASE_NAME}-main"
                    def url = "http://${INSTANCE_IP}:80"

                    echo "Deploying main branch to production at ${url}"

                    // Stop previous container if exists
                    sh "docker stop ${containerName} || true"
                    sh "docker rm ${containerName} || true"

                    // Run production container on port 80
                    sh """
                        docker run -d \
                        --name ${containerName} \
                        -p 80:80 \
                        ${IMAGE_BASE_NAME}:${IMAGE_TAG}
                    """

                    echo "✅ Main branch deployed at ${url}"
                }
            }
        }
    }

    post {
        success { echo "✅ Build and deployment successful!" }
        failure { echo "❌ Deployment failed. Check logs above." }
    }
}
