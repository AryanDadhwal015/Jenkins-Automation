import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        BUILD_DIR = 'dist'
        IMAGE_BASE_NAME = 'my-app'
        CONTAINER_BASE_NAME = 'my-app'
        INSTANCE_IP = '15.206.80.224'
        GIT_REPO_URL = 'https://github.com/AryanDadhwal015/Jenkins-Automation.git'
        PR_PORT = '8085'
        MAIN_PORT = '80'
    }

    stages {
        stage('Check PR') {
            steps {
                script {
                    if (!env.CHANGE_ID) {
                        echo "This is not a PR. Skipping PR preview."
                    } else {
                        echo "PR detected: ${env.CHANGE_ID}"
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
            steps {
                script {
                    def sanitizedBranch = (env.CHANGE_BRANCH ?: 'main').replaceAll('[^a-zA-Z0-9_.-]', '-').toLowerCase()
                    env.SANITIZED_BRANCH = sanitizedBranch
                    env.IMAGE_TAG = "${sanitizedBranch}-${env.BUILD_NUMBER}"

                    echo "Building Docker image: ${IMAGE_BASE_NAME}:${IMAGE_TAG}"
                    sh "docker build -t ${IMAGE_BASE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Deploy PR Preview') {
            when { expression { env.CHANGE_ID != null } }
            steps {
                script {
                    def containerName = "${CONTAINER_BASE_NAME}-pr-${env.SANITIZED_BRANCH}"
                    def url = "http://${INSTANCE_IP}:${PR_PORT}"

                    // Stop and remove previous PR container safely
                    def prevContainers = sh(script: "docker ps -aq -f name=${containerName}", returnStdout: true).trim().split("\\r?\\n")
                    for (c in prevContainers) {
                        if (c) {
                            sh "docker stop ${c} || true"
                            sh "docker rm ${c} || true"
                        }
                    }

                    echo "Starting PR preview container ${containerName} on port ${PR_PORT}"
                    sh """
                        docker run -d \
                        --name ${containerName} \
                        -p ${PR_PORT}:80 \
                        ${IMAGE_BASE_NAME}:${IMAGE_TAG}
                    """

                    echo "✅ PR preview started at ${url}"

                    // Post preview URL back to GitHub PR
                    withCredentials([string(credentialsId: 'GITHUB_PR_TOKEN', variable: 'TOKEN')]) {
                        def repoOwner = 'AryanDadhwal015'
                        def repoName = 'Jenkins-Automation'
                        def commentBody = """
                        🚀 **PR Preview Environment Ready!**
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

        stage('Deploy Main Branch') {
            when { branch 'main' }
            steps {
                script {
                    def containerName = "${CONTAINER_BASE_NAME}-main"
                    def url = "http://${INSTANCE_IP}:${MAIN_PORT}"

                    echo "Deploying main branch to production at ${url}"

                    // Stop previous container safely
                    def prevContainers = sh(script: "docker ps -aq -f name=${containerName}", returnStdout: true).trim().split("\\r?\\n")
                    for (c in prevContainers) {
                        if (c) {
                            sh "docker stop ${c} || true"
                            sh "docker rm ${c} || true"
                        }
                    }

                    // Run production container on port 80
                    sh """
                        docker run -d \
                        --name ${containerName} \
                        -p ${MAIN_PORT}:80 \
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
