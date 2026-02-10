pipeline {
    agent any

    environment {
        BACKEND_PATH = "/var/lib/jenkins/projects/NodeJS-app-deploy-Jenkins/backend"
        FRONTEND_PATH = "/var/lib/jenkins/projects/NodeJS-app-deploy-Jenkins/frontend"
        BUILD_PATH = "/var/www/html/yourapp"
    }

    stages {
        stage('Install Backend Dependencies') {
            steps {
                dir("${BACKEND_PATH}") {
                    sh 'npm install'
                }
            }
        }

        stage('Install Frontend Dependencies') {
            steps {
                dir("${FRONTEND_PATH}") {
                    sh 'npm install'
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir("${FRONTEND_PATH}") {
                    sh 'sudo npm run build'
                }
            }
        }

        stage('Deploy Frontend to Nginx') {
            steps {
                sh "sudo rm -rf ${BUILD_PATH}/*"
                sh "sudo cp -r ${FRONTEND_PATH}/build/* ${BUILD_PATH}/"
            }
        }

        stage('Restart Backend') {
            steps {
                echo "Restarting Node.js backend..."
                sh """
                pm2 restart my-backend || pm2 start ${BACKEND_PATH}/index.js --name my-backend
                pm2 save
                """
            }
        }
    }

    post {
        success {
            echo 'Deployment Successful!'
        }
        failure {
            echo 'Deployment Failed!'
        }
    }
}
