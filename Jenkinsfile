import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        // --- Deployment Variables ---
        SERVER_IP = '13.201.117.191'
        IMAGE_BASE_NAME = 'my-app'
        CONTAINER_BASE_NAME = 'my-app'

        // --- SCM/PR Variables (Set automatically by Multibranch Pipeline) ---
        // Fallback PR_ID to empty string if not a PR build
        PR_ID = "${env.CHANGE_ID ?: ''}" 
        
        // Calculate a unique port for preview environments (starts from 10001)
        DEPLOY_PORT = "${env.BUILD_NUMBER ? 10000 + env.BUILD_NUMBER.toInteger() : 30000}"
        
        // Placeholder for the final URL
        DEPLOY_URL = ''
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
                    // Create a sanitized tag for the image
                    def branchName = env.BRANCH_NAME ?: 'local'
                    def sanitizedBranch = branchName.replaceAll('[^a-zA-Z0-9_.-]', '-').replaceAll('/', '-').toLowerCase()
                    def imageTag = "${sanitizedBranch}-${env.BUILD_NUMBER ?: '0'}"

                    echo "Building Docker image: ${env.IMAGE_BASE_NAME}:${imageTag}"
                    sh "docker build -t ${env.IMAGE_BASE_NAME}:${imageTag} ."
                    
                    env.IMAGE_TAG = imageTag
                }
            }
        }

        // ---------------------------------------------------------------------
        // STAGE 3: DEPLOY PREVIEW ENVIRONMENT (ONLY ON PR BUILD)
        // ---------------------------------------------------------------------
        stage('Deploy Preview Container') {
            when {
                expression { return env.PR_ID != '' } // Only runs if PR_ID (CHANGE_ID) is set
            }
            steps {
                script {
                    def containerName = "${env.CONTAINER_BASE_NAME}-pr-${env.PR_ID}"
                    
                    // 1. Set the final access URL (using the unique port)
                    env.DEPLOY_URL = "http://${env.SERVER_IP}:${env.DEPLOY_PORT}"

                    // 2. Check, Stop, and REMOVE previous container for this PR
                    def previousContainer = sh(
                        script: "docker ps -aq -f name=${containerName}",
                        returnStdout: true,
                        // quiet: true // Remove 'quiet: true' to avoid the Jenkins WARNING
                    ).trim()

                    if (previousContainer) {
                        echo "Stopping and removing previous container: ${previousContainer}"
                        sh "docker stop ${previousContainer}"
                        sh "docker rm ${previousContainer}" // <-- ADDED BACK 'docker rm' for cleanup
                    }

                    // 3. Start new container with unique host port mapped to internal port 80
                    echo "Starting new container '${containerName}' on port ${env.DEPLOY_PORT}..."
                    sh """
                      docker run -d \\
                        --name ${containerName} \\
                        -p ${env.DEPLOY_PORT}:80 \\ 
                        ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG}
                    """
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
                    expression { return env.PR_ID != '' }
                    expression { return currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                }
            }
            steps {
                script {
                    // Securely get the GitHub token from Jenkins Credentials
                    withCredentials([string(credentialsId: 'GITHUB_PR_TOKEN', variable: 'TOKEN')]) {
                        
                        // --- REPO NAMES (MUST BE CLEAN, NO SPACES) ---
                        def repoOwner = 'AryanDadhwal015'
                        def repoName = 'Jenkins-Automation'
                        
                        // Create the multi-line comment body
                        def commentBody = """
                        🎉 **Preview Environment Ready!** 🎉
                        
                        The changes in this PR have been deployed for review.
                        
                        **Access URL:** ${env.DEPLOY_URL}
                        
                        **Image Tag:** ${env.IMAGE_BASE_NAME}:${env.IMAGE_TAG}
                        """

                        // Safely convert the comment body into a JSON string
                        def jsonPayload = JsonOutput.toJson([body: commentBody])
                        
                        // Execute curl command (using secure, escaped interpolation)
                        sh """
                            curl -i -X POST \\
                              -H "Authorization: token \\\$TOKEN" \\ // SECURE: Groovy ignores, Bash interpolates
                              -H "Content-Type: application/json" \\
                              -d '${jsonPayload}' \\               // SAFE: JSON payload wrapped in single quotes
                              "https://api.github.com/repos/${repoOwner}/${repoName}/issues/${env.PR_ID}/comments"
                        """
                    }
                }
            }
        }

        // ---------------------------------------------------------------------
        // STAGE 5: CLEAN UP/MAIN DEPLOYMENT (ONLY ON NON-PR BUILD)
        // ---------------------------------------------------------------------
        stage('Clean Up Non-PR Deployment') {
            when {
                expression { return env.PR_ID == '' } // Only runs if PR_ID is NOT set (e.g., merge to main)
            }
            steps {
                script {
                    // Standard cleanup and deployment to port 80 for main branch
                    def previousContainer = sh(
                        script: "docker ps -aq -f name=${env.CONTAINER_BASE_NAME}",
                        returnStdout: true,
                        // quiet: true // Remove 'quiet: true' to avoid the Jenkins WARNING
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