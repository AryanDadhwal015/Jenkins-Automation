// NOTE: This assumes your Jenkins setup provides PR info via the 'CHANGE_ID' environment variable.
// It also assumes you have a secret credential named 'GITHUB_PR_TOKEN' for posting the comment

pipeline {
    agent any

    environment {
        // --- Existing Variables ---
        DEPLOY_METHOD = "${env.DEPLOY_METHOD ?: ''}"
        BUILD_DIR = 'dist'
        IMAGE_BASE_NAME = 'my-app'
        CONTAINER_BASE_NAME = 'my-app'
        SERVER_IP = '13.201.117.191' // <-- The IP where the Jenkins agent/Docker host is running

        // --- New Variables for PR Deployment ---
        PR_ID = "${env.CHANGE_ID ?: ''}" // Get Pull Request ID (if building a PR)
        // Use the BUILD_NUMBER for a unique port mapping (e.g., 10001, 10002, etc.)
        // Default to a high port if BUILD_NUMBER is not set, or ensure it's a number.
        DEPLOY_PORT = "${env.BUILD_NUMBER ? 10000 + env.BUILD_NUMBER.toInteger() : 30000}"
        DEPLOY_URL = '' // Placeholder for the final access URL
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

        stage('Deploy Preview Container') {
            // This stage only runs if we are building a Pull Request
            when {
                expression { return env.PR_ID != '' }
            }
            steps {
                script {
                    def containerName = "${env.CONTAINER_BASE_NAME}-pr-${env.PR_ID}"
                    env.DEPLOY_URL = "http://${env.SERVER_IP}:${env.DEPLOY_PORT}"

                    // 1. Check, Stop, and REMOVE previous container for this PR
                    def previousContainer = sh(
                        script: "docker ps -aq -f name=${containerName}",
                        returnStdout: true,
                        quiet: true
                    ).trim()

                    if (previousContainer) {
                        echo "Stopping and removing previous container: ${previousContainer}"
                        sh "docker stop ${previousContainer}"
                        sh "docker rm ${previousContainer}"
                    }

                    // 2. Start new container with unique port mapping
                    echo "Starting new container '${containerName}' using unique port ${env.DEPLOY_PORT}..."
                    sh """
                      docker run -d \\
                        --name ${containerName} \\
                        -p ${env.DEPLOY_PORT}:80 \\
                        ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG}
                    """

                    echo "Preview application deployed successfully!"
                    echo "Access URL: ${env.DEPLOY_URL}"
                }
            }
        }
        
        stage('Notify Pull Request') {
            // This stage runs only if it was a PR build AND the deployment succeeded
            when {
                allOf {
                    expression { return env.PR_ID != '' }
                    // Only post comment if the stage before it was successful
                    expression { return currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                }
            }
            steps {
                script {
                    // Securely get the GitHub token from Jenkins Credentials
                    withCredentials([string(credentialsId: 'GITHUB_PR_TOKEN', variable: 'TOKEN')]) {
                        
                        def repoOwner = 'AryanDadhwal015  ' // <<--- CHANGE ME
                        def repoName = 'Jenkins-Automation'   // <<--- CHANGE ME
                        
                        def commentBody = """
                        🎉 **Preview Environment Ready!** 🎉
                        
                        The changes in this PR have been deployed for review.
                        
                        **Access URL:** ${env.DEPLOY_URL}
                        
                        **Image Tag:** ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG}
                        """
                        
                        // Use the GitHub API to post the comment
                        sh """
                        curl -i -X POST \
                          -H "Authorization: token ${TOKEN}" \
                          -d '{"body": "${commentBody}"}' \
                          "https://api.github.com/repos/${repoOwner}/${repoName}/issues/${env.PR_ID}/comments"
                        """
                    }
                }
            }
        }

        stage('Clean Up Non-PR Deployment (Original Logic)') {
            // This stage runs if it was NOT a Pull Request build (e.g., a main branch build)
            when {
                expression { return env.PR_ID == '' }
            }
            steps {
                script {
                    // This is your original deployment logic for single container on port 80
                    def previousContainer = sh(
                        script: "docker ps -aq -f name=${env.CONTAINER_BASE_NAME}",
                        returnStdout: true,
                        quiet: true
                    ).trim()

                    if (previousContainer) {
                        echo "Stopping and removing previous container: ${previousContainer}"
                        sh "docker stop ${previousContainer}"
                        sh "docker rm ${previousContainer}"
                    }

                    echo "Starting new container with image ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG} on port 80"
                    sh """
                        docker run -d \\
                            --name ${env.CONTAINER_BASE_NAME} \\
                            -p 80:80 \\
                            ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG}
                    """

                    echo "Production container deployed successfully on http://${env.SERVER_IP}:80"
                }
            }
        }
    }
}
