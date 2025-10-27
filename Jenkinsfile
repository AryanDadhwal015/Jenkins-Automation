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
            when { expression { env.CHANGE_ID != null } }  // Only for PRs
            steps {
                script {
                    def containerName = "${CONTAINER_BASE_NAME}-pr"
                    def url = "http://${INSTANCE_IP}:${PR_PREVIEW_PORT}"

                    echo "Deploying PR #${env.CHANGE_ID} preview at ${url}"

                    // Stop previous container if exists
