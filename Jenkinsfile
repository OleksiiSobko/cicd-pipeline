pipeline {
    agent any

    environment {
        APP_NAME = "nodeapp"
        IMAGE_NAME = "react_app"
        REGISTRY = "local"
    }

    tools {
        nodejs 'node7'
    }

    stages {

        stage('Setup Environment') {
            steps {
                script {
                    env.APP_NAME = (env.BRANCH_NAME == 'main') ? 'nodemain' : 'nodedev'
                    env.APP_PORT = (env.BRANCH_NAME == 'main') ? '3000' : '3001'
                    echo "Environment configured for branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Install & Test') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'main'
                    changeRequest()
                }
            }
            steps {
                echo "Installing dependencies..."
                sh 'npm install'

                echo "Running tests..."
                sh 'npm test || true'
            }
        }

        stage('Build Docker Image') {
            when {
                branch pattern: "^(dev|main)$", comparator: "REGEXP"
            }
            steps {
                script {
                    sh """
                    docker build -t ${IMAGE_NAME}:${env.BUILD_ID} .
                    """
                }
            }
        }

        stage('Scan Docker Image for Vulnerabilities') {
            when {
                branch pattern: "^(dev|main)$", comparator: "REGEXP"
            }
            steps {
                script {
                    def vulnerabilities = sh(
                        script: "trivy image --exit-code 0 --severity HIGH,MEDIUM,LOW --no-progress ${IMAGE_NAME}:${env.BUILD_ID}",
                        returnStdout: true
                    ).trim()
                    echo "Vulnerability Report:\\n${vulnerabilities}"
                }
            }
        }

        stage('Deploy') {
            when {
                branch pattern: "^(dev|main)$", comparator: "REGEXP"
            }
            steps {
                script {
                    echo "Cleaning up old containers for env: ${env.BRANCH_NAME}"
                    sh """
                    docker ps -a --filter "name=${APP_NAME}-${env.BRANCH_NAME}" -q | xargs -r docker stop
                    docker ps -a --filter "name=${APP_NAME}-${env.BRANCH_NAME}" -q | xargs -r docker rm

                    echo "Starting new container on port ${APP_PORT}"
                    docker run -d \
                        --name ${APP_NAME}-${env.BRANCH_NAME} \
                        --label env=${env.BRANCH_NAME} \
                        -p ${APP_PORT}:3000 ${IMAGE_NAME}:${env.BUILD_ID}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Successfully deployed ${env.BRANCH_NAME} on port ${env.APP_PORT}"
        }
        failure {
            echo "Build failed for ${env.BRANCH_NAME}"
        }
    }
}
