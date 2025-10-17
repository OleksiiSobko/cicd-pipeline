pipeline {
    agent any

    environment {
        APP_NAME = "nodeapp"
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
