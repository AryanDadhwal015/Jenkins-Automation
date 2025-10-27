pipeline {
    agent any

    environment {
        // Ensure a sensible default so preview/dev SCP deploys run when DEPLOY_METHOD isn't set in Jenkins
        DEPLOY_METHOD = "${env.DEPLOY_METHOD ?: ''}"
        BUILD_DIR = 'dist'
        // Image tag uses branch name and build number when available
        IMAGE_TAG = "${env.BRANCH_NAME ?: 'local'}-${env.BUILD_NUMBER ?: '0'}"
        IMAGE_BASE_NAME = 'my-app'
        CONTAINER_BASE_NAME = 'my-app'
        SERVER_IP = '172.31.76.29'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // stage('Print Config') {
        //   steps {
        //     withCredentials([
        //       string(credentialsId: 'preview-path', variable: 'PREVIEW_PATH'),
        //       string(credentialsId: 'dev-deploy-path', variable: 'DEV_DEPLOY_PATH'),
        //       string(credentialsId: 'preview-server', variable: 'PREVIEW_SERVER')
        //     ]) {
        //       echo "Branch: ${env.BRANCH_NAME ?: 'local'}"
        //       echo "DEPLOY_METHOD: ${env.DEPLOY_METHOD}"
        //       echo "PREVIEW_PATH (from credentials): ${PREVIEW_PATH}"
        //       echo "DEV_DEPLOY_PATH (from credentials): ${DEV_DEPLOY_PATH}"
        //       echo "PREVIEW_SERVER (from credentials): ${PREVIEW_SERVER}"
        //     }
        //   }
        // }

        stage('Build Docker Image') {
            steps {
                script {
                    // Sanitize branch name for Docker tag and container usage
                    def branch = env.BRANCH_NAME ?: env.CHANGE_BRANCH ?: 'local'
                    def sanitizedBranch = branch.replaceAll('[^a-zA-Z0-9_.-]', '-').toLowerCase()
                    env.SANITIZED_BRANCH = sanitizedBranch
                    env.IMAGE_TAG = "${sanitizedBranch}-${env.BUILD_NUMBER}"

                    echo "Building Docker image: ${IMAGE_BASE_NAME}:${IMAGE_TAG}"
                    sh "docker build -t ${IMAGE_BASE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Deploy Container') {
            steps {
                script {
                    // Use sanitized branch in container name
                    def containerName = "${CONTAINER_BASE_NAME}-${env.SANITIZED_BRANCH}"
                    def url = "http://${SERVER_IP}:${HOST_PORT}"

                    // Stop previous container if exists
                    def previousContainer = sh(script: "docker ps -aq -f name=${containerName}", returnStdout: true).trim()
                    if (previousContainer) {
                        echo "Stopping previous container: ${previousContainer}"
                        sh "docker stop ${previousContainer}"
                        sh "docker rm ${previousContainer}"
                    }

                    // Run new container
                    echo "Starting new container ${containerName}"
                    sh """
                        docker run -d \
                          --name ${containerName} \
                          -p ${HOST_PORT}:${CONTAINER_PORT} \
                          ${IMAGE_BASE_NAME}:${IMAGE_TAG}
                    """
                    echo "✅ Container started at ${url}"

                    // Post PR comment if this is a pull request
                    if (env.CHANGE_ID) {
                        withCredentials([string(credentialsId: 'GITHUB_PR_TOKEN', variable: 'TOKEN')]) {       // <-- Set the Credential ID here as specified inside Jenkins Credentials
                            def repoOwner = 'AryanDadhwal015'                                                        // <-- Change it to your GitHub username
                            def repoName = 'Jenkins-Automation'                                                        // <-- Change it to your repository name
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

        // stage('Archive Build Artifacts') {
        //     steps {
        //         archiveArtifacts artifacts: "${BUILD_DIR}/**", fingerprint: true
        //     }
        // }

        // SCP-based preview (default) - runs when DEPLOY_METHOD != 'docker' and branch != dev
        // stage('Deploy Preview (for PRs and feature branches)') {
        //   when {
        //     expression { return (env.BRANCH_NAME ?: '') != 'dev' && (env.DEPLOY_METHOD ?: 'scp') != 'docker' }
        //   }
        //   steps {
        //     withCredentials([
        //       sshUserPrivateKey(credentialsId: 'ec2-ssh', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
        //       string(credentialsId: 'preview-server', variable: 'PREVIEW_SERVER'),
        //       string(credentialsId: 'preview-path', variable: 'PREVIEW_PATH')
        //     ]) {
        //       echo "Deploying preview for ${env.BRANCH_NAME} via SCP..."
        //       sh '''
        //         PREVIEW_DIR=\${PREVIEW_PATH}/previews/\${BRANCH_NAME}
        //         ssh -i $SSH_KEY -o StrictHostKeyChecking=no $SSH_USER@$PREVIEW_SERVER "mkdir -p $PREVIEW_DIR"
        //         scp -i $SSH_KEY -o StrictHostKeyChecking=no -r \${BUILD_DIR}/* $SSH_USER@$PREVIEW_SERVER:$PREVIEW_DIR/
        //         ssh -i $SSH_KEY -o StrictHostKeyChecking=no $SSH_USER@$PREVIEW_SERVER "ls -la $PREVIEW_DIR | head -n 20; [ -f $PREVIEW_DIR/index.html ] && echo 'index.html exists' || echo 'index.html missing'"
        //       '''
        //       echo "🌐 Preview available at: https://${PREVIEW_SERVER}/previews/${env.BRANCH_NAME}/"
        //     }
        //   }
        // }

        // SCP-based dev deploy (default)
        // stage('Deploy to Dev') {
        //   when {
        //     expression {
        //       def bn = (env.BRANCH_NAME ?: '')
        //       return (bn.tokenize('/').last() == 'dev') && ((env.DEPLOY_METHOD ?: 'scp') != 'docker')
        //     }
        //   }
        //   steps {
        //     withCredentials([
        //       sshUserPrivateKey(credentialsId: 'ec2-ssh', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
        //       string(credentialsId: 'preview-server', variable: 'PREVIEW_SERVER'),
        //       string(credentialsId: 'dev-deploy-path', variable: 'DEV_DEPLOY_PATH')
        //     ]) {
        //       echo 'Deploying dev build via SCP...'
        //       sh '''
        //         ssh -i $SSH_KEY -o StrictHostKeyChecking=no $SSH_USER@$PREVIEW_SERVER "mkdir -p $DEV_DEPLOY_PATH"
        //         scp -i $SSH_KEY -o StrictHostKeyChecking=no -r \${BUILD_DIR}/* $SSH_USER@$PREVIEW_SERVER:$DEV_DEPLOY_PATH/
        //       '''
        //       echo "✅ Dev environment deployed at: https://${PREVIEW_SERVER}/dev/"
        //     }
        //   }
        // }

        // Docker-based deploy (optional). To use this, set DEPLOY_METHOD=docker in the Jenkins job or pipeline.
        // stage('Docker: Build Image') {
        //   when {
        //     expression { return (env.DEPLOY_METHOD ?: 'scp') == 'docker' }
        //   }
        //   steps {
        //     echo 'Building Docker image...'
        //     sh 'docker --version || true'
        //     sh '''
        //       IMAGE_NAME=frontend:\${IMAGE_TAG}
        //       docker build -t $IMAGE_NAME . 
        //       docker save $IMAGE_NAME -o app-image.tar
        //     '''
        //     archiveArtifacts artifacts: 'app-image.tar', fingerprint: true
        //   }
        // }

        // stage('Docker: Deploy to Server') {
        //   when {
        //     expression { return (env.DEPLOY_METHOD ?: 'scp') == 'docker' }
        //   }
        //   steps {
        //     withCredentials([
        //       sshUserPrivateKey(credentialsId: 'ec2-ssh', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
        //       string(credentialsId: 'preview-server', variable: 'PREVIEW_SERVER'),
        //       string(credentialsId: 'preview-path', variable: 'PREVIEW_PATH'),
        //       string(credentialsId: 'dev-deploy-path', variable: 'DEV_DEPLOY_PATH')
        //     ]) {
        //       echo 'Transferring Docker image to server and running container...'
        //       sh '''
        //         TARGET_DIR=\${PREVIEW_PATH}/previews/\${BRANCH_NAME}
        //         ssh -i $SSH_KEY -o StrictHostKeyChecking=no $SSH_USER@$PREVIEW_SERVER "mkdir -p $TARGET_DIR"
        //         scp -i $SSH_KEY -o StrictHostKeyChecking=no app-image.tar $SSH_USER@$PREVIEW_SERVER:$TARGET_DIR/
        //         ssh -i $SSH_KEY -o StrictHost
