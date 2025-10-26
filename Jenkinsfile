import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        // --- Deployment Variables ---
        SERVER_IP = '13.233.134.67'
        IMAGE_BASE_NAME = 'my-app'
        CONTAINER_BASE_NAME = 'my-app'

        // --- SCM/PR Variables (Set automatically by Multibranch Pipeline) ---
        // Fallback PR_ID to 'none' if not a PR build
        PR_ID = "${env.CHANGE_ID ?: 'none'}"

        // Calculate a unique port for preview environments (starts from 10001)
        DEPLOY_PORT = "${env.BUILD_NUMBER ? 10000 + env.BUILD_NUMBER.toInteger() : 30000}"

        // Placeholder for the final URL
        DEPLOY_URL = ''
        IMAGE_TAG = ''
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // ---------------------------------------------------------------------
        // STAGE 2: BUILD DOCKER IMAGE
        // ---------------------------------------------------------------------
        stage('Build Docker Image') {
            steps {
                script {
                    def branchName = env.BRANCH_NAME ?: 'local'
                    def sanitizedBranch = branchName
                        .replaceAll('[^a-zA-Z0-9_.-]', '-')
                        .replaceAll('/', '-')
                        .toLowerCase()
                    def imageTag = "${sanitizedBranch}-${env.BUILD_NUMBER ?: '0'}"

                    echo "Building Docker image: ${env.IMAGE_BASE_NAME}:${imageTag}"
                    sh "docker build -t ${env.IMAGE_BASE_NAME}:${imageTag} ."

                    // Persist image tag globally
                    env.IMAGE_TAG = imageTag
                }
            }
        }

        // ---------------------------------------------------------------------
        // STAGE 3: DEPLOY PREVIEW CONTAINER (ONLY FOR PR BUILDS)
        // ---------------------------------------------------------------------
        stage('Deploy Preview Container') {
            when {
                expression { return env.PR_ID != 'none' } // Only run for PR builds
            }
            steps {
                script {
                    def containerName = "${env.CONTAINER_BASE_NAME}-pr-${env.PR_ID}"

                    echo "DEBUG: CHANGE_ID=${env.CHANGE_ID}, PR_ID=${env.PR_ID}, BRANCH=${env.BRANCH_NAME}"
                    echo "Starting new container '${containerName}' on port ${env.DEPLOY_PORT}..."

                    // Set final access URL
                    env.DEPLOY_URL = "http://${env.SERVER_IP}:${env.DEPLOY_PORT}"

                    // Stop and remove previous container (if any)
                    def previousContainer = sh(
                        script: "docker ps -aq -f name=${containerName}",
                        returnStdout: true
                    ).trim()

                    if (previousContainer) {
                        echo "Stopping and removing previous container: ${previousContainer}"
                        sh "docker stop ${previousContainer}"
                        sh "docker rm ${previousContainer}"
                    }

                    // Start new container
                    sh """
                        docker run -d \\
                            --name ${containerName} \\
                            -p ${env.DEPLOY_PORT}:80 \\
                            ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG}
                    """

                    echo "✅ Preview deployed successfully!"
                    echo "Access URL: ${env.DEPLOY_URL}"
                }
            }
        }

        // ---------------------------------------------------------------------
        // STAGE 4: NOTIFY PULL REQUEST (ONLY ON SUCCESSFUL PR DEPLOYMENT)
        // ---------------------------------------------------------------------
        stage('Notify Pull Request') {
            when {
                allOf {
                    expression { return env.PR_ID != 'none' }
                    expression { return currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                }
            }
            steps {
                script {
                    withCredentials([string(credentialsId: 'GITHUB_PR_TOKEN', variable: 'TOKEN')]) {
                        def repoOwner = 'AryanDadhwal015'
                        def repoName = 'Jenkins-Automation'

                        def commentBody = """
                        🎉 **Preview Environment Ready!** 🎉

                        The changes in this PR have been deployed for review.

                        **Access URL:** ${env.DEPLOY_URL}

                        **Image Tag:** ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG}
                        """

                        def jsonPayload = JsonOutput.toJson([body: commentBody])

                        sh """
                            curl -i -X POST \\
                              -H "Authorization: token \${TOKEN}" \\
                              -H "Content-Type: application/json" \\
                              -d '${jsonPayload}' \\
                              "https://api.github.com/repos/${repoOwner}/${repoName}/issues/${env.PR_ID}/comments"
                        """
                    }
                }
            }
        }

        // ---------------------------------------------------------------------
        // STAGE 5: CLEAN UP / MAIN DEPLOYMENT (FOR NON-PR BUILDS)
        // ---------------------------------------------------------------------
        stage('Clean Up Non-PR Deployment') {
            when {
                expression { return env.PR_ID == 'none' }
            }
            steps {
                script {
                    def previousContainer = sh(
                        script: "docker ps -aq -f name=${env.CONTAINER_BASE_NAME}",
                        returnStdout: true
                    ).trim()

                    if (previousContainer) {
                        echo "Stopping and removing previous container: ${previousContainer}"
                        sh "docker stop ${previousContainer}"
                        sh "docker rm ${previousContainer}"
                    }

                    echo "Starting production container on http://${env.SERVER_IP}:80"
                    sh """
                        docker run -d \\
                            --name ${env.CONTAINER_BASE_NAME} \\
                            -p 80:80 \\
                            ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG}
                    """
                }
            }
        }
    }
}
