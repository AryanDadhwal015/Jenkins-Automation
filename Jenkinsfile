import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        BUILD_DIR = 'dist'
        IMAGE_BASE_NAME = 'my-app'
        CONTAINER_BASE_NAME = 'my-app'
        INSTANCE_IP = '15.206.80.224'
        CONTAINER_PORT = '80'
        PR_HOST_PORT = '8085'
        MAIN_HOST_PORT = '80'
        GIT_REPO_URL = 'https://github.com/AryanDadhwal015/Jenkins-Automation.git'
    }

    stages {
        stage('Check PR') {
            steps {
                script {
                    if (!env.CHANGE_ID) {
                        echo "This is not a PR. Skipping PR build."
                    } else {
                        echo "PR detected: #${env.CHANGE_ID} on branch ${env.CHANGE_BRANCH}"
                    }
                }
            }
        }

        stage('Clone from GitHub') {
            steps {
                script {
                    def branchToCheckout = env.CHANGE_ID ? env.CHANGE_BRANCH : env.BRANCH_NAME
                    echo "Cloning ${GIT_REPO_URL} (branch: ${branchToCheckout}) ..."
                    checkout([$class: 'GitSCM',
                        branches: [[name: "${branchToCheckout}"]],
                        userRemoteConfigs: [[url: "${GIT_REPO_URL}"]]
                    ])
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def branchTag = env.CHANGE_ID ? "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}" : "main-${env.BUILD_NUMBER}"
                    env.IMAGE_TAG = branchTag
                    echo "Building Docker image: ${IMAGE_BASE_NAME}:${IMAGE_TAG}"
                    sh "docker build -t ${IMAGE_BASE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Deploy PR Preview') {
            when { expression { env.CHANGE_ID != null } }
            steps {
                script {
                    def containerName = "${CONTAINER_BASE_NAME}-pr-${env.CHANGE_ID}"
                    def url = "http://${INSTANCE_IP}:${PR_HOST_PORT}"

                    // Stop and remove previous PR container if exists
                    sh "docker rm -f ${containerName} || true"

                    // Run new PR container
                    echo "Starting PR container ${containerName} on port ${PR_HOST_PORT}"
                    sh """
                        docker run -d \
                        --name ${containerName} \
                        -p ${PR_HOST_PORT}:${CONTAINER_PORT} \
                        ${IMAGE_BASE_NAME}:${IMAGE_TAG}
                    """
                    echo "✅ PR Preview deployed at ${url}"

                    // Post preview link back to PR
                    withCredentials([string(credentialsId: 'GITHUB_PR_TOKEN', variable: 'TOKEN')]) {
                        def repoOwner = 'AryanDadhwal015'
                        def repoName = 'Jenkins-Automation'
                        def commentBody = """
                        🚀 **Preview Environment Ready!**
                        - **URL:** ${url}
                        - **Image:** ${IMAGE_BASE_NAME}:${IMAGE_TAG}
                        - **Branch:** ${env.CHANGE_BRANCH}
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
                    def url = "http://${INSTANCE_IP}:${MAIN_HOST_PORT}"

                    echo "Deploying main branch to production at ${url}"

                    // Stop previous main container if exists
                    sh "docker rm -f ${containerName} || true"

                    // Run production container
                    sh """
                        docker run -d \
                        --name ${containerName} \
                        -p ${MAIN_HOST_PORT}:${CONTAINER_PORT} \
                        ${IMAGE_BASE_NAME}:${IMAGE_TAG}
                    """
                    echo "✅ Main branch deployed at ${url}"
                }
            }
        }
    }

    post {
        success { echo "✅ Deployment successful!" }
        failure { echo "❌ Deployment failed. Check logs above." }
    }
}
