import groovy.json.JsonOutput

pipeline {
    agent any

    environment {
        SERVER_IP = '13.233.134.67'
        IMAGE_BASE_NAME = 'my-app'
        CONTAINER_BASE_NAME = 'my-app'
        PR_ID = "${env.CHANGE_ID ?: 'none'}"
        DEPLOY_PORT = "${env.BUILD_NUMBER ? 10000 + env.BUILD_NUMBER.toInteger() : 30000}"
        DEPLOY_URL = ''
        IMAGE_TAG_FILE = 'image_tag.txt' // <--- New: temp file to store tag persistently
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
                    def branchName = env.BRANCH_NAME ?: 'local'
                    def sanitizedBranch = branchName.replaceAll('[^a-zA-Z0-9_.-]', '-')
                                                    .replaceAll('/', '-')
                                                    .toLowerCase()
                    def imageTag = "${sanitizedBranch}-${env.BUILD_NUMBER ?: '0'}"

                    echo "Building Docker image: ${env.IMAGE_BASE_NAME}:${imageTag}"
                    sh "docker build -t ${env.IMAGE_BASE_NAME}:${imageTag} ."

                    // Write tag to file for persistence between stages
                    writeFile file: env.IMAGE_TAG_FILE, text: imageTag
                }
            }
        }

        stage('Deploy Preview Container') {
            when {
                expression { return env.PR_ID != 'none' }
            }
            steps {
                script {
                    // Read the persisted tag
                    def imageTag = readFile(env.IMAGE_TAG_FILE).trim()
                    def containerName = "${env.CONTAINER_BASE_NAME}-pr-${env.PR_ID}"
                    env.DEPLOY_URL = "http://${env.SERVER_IP}:${env.DEPLOY_PORT}"

                    echo "Starting new PR container '${containerName}' using image ${env.IMAGE_BASE_NAME}:${imageTag}"

                    def previousContainer = sh(script: "docker ps -aq -f name=${containerName}", returnStdout: true).trim()
                    if (previousContainer) {
                        sh "docker stop ${previousContainer}"
                        sh "docker rm ${previousContainer}"
                    }

                    sh """
                        docker run -d \\
                            --name ${containerName} \\
                            -p ${env.DEPLOY_PORT}:80 \\
                            ${env.IMAGE_BASE_NAME}:${imageTag}
                    """

                    echo "✅ Preview deployed at ${env.DEPLOY_URL}"
                }
            }
        }

        stage('Notify Pull Request') {
            when {
                allOf {
                    expression { return env.PR_ID != 'none' }
                    expression { return currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                }
            }
            steps {
                script {
                    def imageTag = readFile(env.IMAGE_TAG_FILE).trim()
                    withCredentials([string(credentialsId: 'GITHUB_PR_TOKEN', variable: 'TOKEN')]) {
                        def repoOwner = 'AryanDadhwal015'
                        def repoName = 'Jenkins-Automation'

                        def commentBody = """
                        🎉 **Preview Environment Ready!**

                        **Access URL:** ${env.DEPLOY_URL}

                        **Image Tag:** ${env.IMAGE_BASE_NAME}:${imageTag}
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

        stage('Clean Up Non-PR Deployment') {
            when {
                expression { return env.PR_ID == 'none' }
            }
            steps {
                script {
                    // Read the persisted tag
                    def imageTag = readFile(env.IMAGE_TAG_FILE).trim()

                    def previousContainer = sh(script: "docker ps -aq -f name=${env.CONTAINER_BASE_NAME}", returnStdout: true).trim()
                    if (previousContainer) {
                        sh "docker stop ${previousContainer}"
                        sh "docker rm ${previousContainer}"
                    }

                    echo "Starting production container using image: ${env.IMAGE_BASE_NAME}:${imageTag}"
                    sh """
                        docker run -d \\
                            --name ${env.CONTAINER_BASE_NAME} \\
                            -p 80:80 \\
                            ${env.IMAGE_BASE_NAME}:${imageTag}
                    """
                    echo "✅ App deployed on http://${env.SERVER_IP}:80"
                }
            }
        }
    }
}
