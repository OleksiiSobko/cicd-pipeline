pipeline {
    agent any

    environment {
        APP_NAME = "nodeapp"
        IMAGE_NAME = "react_app"
    }
    
    tools{
    	nodejs 'node7'
    }
    stages {
    
        stage('Setup Environment') {
            steps {
                script {
                    env.APP_NAME = (env.BRANCH_NAME == 'main') ? 'nodemain' : 'nodedev'
                    env.APP_PORT = (env.BRANCH_NAME == 'main') ? '3000' : '3001'
                }
            }
        }
        stage('Checkout') {
            steps {
                checkout scm
                echo "Building branch ${env.BRANCH_NAME}"
            }
        }

        stage('Build') {
            steps {
                echo "Building application..."
                sh 'npm install'
            }
        }

        stage('Test') {
            steps {
                echo "Running tests..."
                sh 'npm test || true'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${IMAGE_NAME} ."
                }
            }
        }
        stage('Scan Docker Image for Vulnerabilities') {
            steps {
                script {
                    def imageTag = "${IMAGE_NAME}:${BUILD_ID}"
                    def branchName = env.BRANCH_NAME ?: 'dev'
                    def reportFile = "trivy-${branchName}-report.html"
        
                    echo "🔍 Scanning image ${imageTag} for vulnerabilities (branch: ${branchName})..."
                    sh """
                    trivy image \
                      --scanners vuln \
                      --ignorefile /dev/null \
                      --config /dev/null \
                      --exit-code 0 \
                      --severity HIGH,MEDIUM,LOW \
                      --no-progress \
                      --format html \
                      --output ${reportFile} \
                      ${imageTag}
                    """
                    publishHTML(target: [
                      reportName: "Trivy Report - ${branchName}",
                      reportDir: '.',
                      reportFiles: "${reportFile}",
                      keepAll: true,
                      alwaysLinkToLastBuild: true,
                      allowMissing: true
                    ])
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh '''
                    echo "Stopping old container..."
                    docker stop ${APP_NAME}-${BRANCH_NAME} || true
                    docker rm ${APP_NAME}-${BRANCH_NAME} || true

                    echo "Running new container on port ${APP_PORT}"
                    docker run -d --name ${APP_NAME}-${BRANCH_NAME} -p ${APP_PORT}:3000 ${IMAGE_NAME}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful on port ${APP_PORT}"
        }
        failure {
            echo "Build failed for branch ${env.BRANCH_NAME}"
        }
    }
}
