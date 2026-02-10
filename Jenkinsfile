pipeline {
    agent any

    environment {
        BACKEND_PATH = "${WORKSPACE}/backend"
        FRONTEND_PATH = "${WORKSPACE}/frontend"
        DEPLOY_SERVER = "ubuntu@YOUR_SERVER_IP" 
        DEPLOY_PATH_BACKEND = "/var/www/backend" 
        DEPLOY_PATH_FRONTEND = "/var/www/frontend" 
        SSH_KEY = "/var/lib/jenkins/.ssh/id_rsa" 
        PM2_APP_NAME = "nodejs-app" // PM2 app name
    }

    options {
        timestamps()
        buildDiscarder(logRotator(daysToKeepStr: '30', numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out source code..."
                checkout scm
            }
        }

        stage('Install Backend Dependencies') {
            steps {
                dir("${BACKEND_PATH}") {
                    echo "Installing backend dependencies..."
                    sh 'npm install'
                }
            }
        }

        stage('Test Backend') {
            steps {
                dir("${BACKEND_PATH}") {
                    echo "Running backend tests..."
                    sh 'npm test || true'
                }
            }
        }

        stage('Build Backend') {
            steps {
                dir("${BACKEND_PATH}") {
                    echo "Building backend..."
                    sh 'npm run build'
                    archiveArtifacts artifacts: 'dist/**', fingerprint: true
                    stash includes: 'dist/**', name: 'backend-dist'
                }
            }
        }

        stage('Install Frontend Dependencies') {
            steps {
                dir("${FRONTEND_PATH}") {
                    echo "Installing frontend dependencies..."
                    sh 'npm install'
                }
            }
        }

        stage('Test Frontend') {
            steps {
                dir("${FRONTEND_PATH}") {
                    echo "Running frontend tests..."
                    sh 'npm test || true'
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir("${FRONTEND_PATH}") {
                    echo "Building frontend..."
                    sh 'npm run build'
                    archiveArtifacts artifacts: 'build/**', fingerprint: true
                    stash includes: 'build/**', name: 'frontend-build'
                }
            }
        }

        stage('Deploy Backend') {
            steps {
                echo "Deploying backend..."
                unstash 'backend-dist'
                sh """
                    ssh -i ${SSH_KEY} ${DEPLOY_SERVER} 'mkdir -p ${DEPLOY_PATH_BACKEND}/backup'
                    ssh -i ${SSH_KEY} ${DEPLOY_SERVER} 'if [ -d ${DEPLOY_PATH_BACKEND}/dist ]; then tar -czf ${DEPLOY_PATH_BACKEND}/backup/backup_\$(date +%F_%T).tar.gz -C ${DEPLOY_PATH_BACKEND} dist; fi'
                    scp -i ${SSH_KEY} -r dist/* ${DEPLOY_SERVER}:${DEPLOY_PATH_BACKEND}/
                    ssh -i ${SSH_KEY} ${DEPLOY_SERVER} 'cd ${DEPLOY_PATH_BACKEND} && npm install --production && pm2 restart ${PM2_APP_NAME} || pm2 start index.js --name ${PM2_APP_NAME}'
                """
            }
        }

        stage('Deploy Frontend') {
            steps {
                echo "Deploying frontend..."
                unstash 'frontend-build'
                sh """
                    ssh -i ${SSH_KEY} ${DEPLOY_SERVER} 'mkdir -p ${DEPLOY_PATH_FRONTEND}/backup'
                    ssh -i ${SSH_KEY} ${DEPLOY_SERVER} 'if [ -d ${DEPLOY_PATH_FRONTEND}/build ]; then tar -czf ${DEPLOY_PATH_FRONTEND}/backup/backup_\$(date +%F_%T).tar.gz -C ${DEPLOY_PATH_FRONTEND} build; fi'
                    scp -i ${SSH_KEY} -r build/* ${DEPLOY_SERVER}:${DEPLOY_PATH_FRONTEND}/
                    ssh -i ${SSH_KEY} ${DEPLOY_SERVER} 'sudo nginx -s reload'
                """
            }
        }
    }

    post {
        success {
            echo "Deployment completed successfully!"
            // Optional: Slack/Email notification
        }
        failure {
            echo "Deployment failed! Triggering rollback..."
            sh """
                ssh -i \${SSH_KEY} \${DEPLOY_SERVER} '
                    BACKUP_FILE=\$(ls -t \${DEPLOY_PATH_BACKEND}/backup/backup_*.tar.gz | head -1)
                if [ -f "\$BACKUP_FILE" ]; then
                    tar -xzf \$BACKUP_FILE -C \${DEPLOY_PATH_BACKEND}
                    pm2 restart \${PM2_APP_NAME}
                fi

                    BACKUP_FRONTEND=\$(ls -t \${DEPLOY_PATH_FRONTEND}/backup/backup_*.tar.gz | head -1)
                if [ -f "\$BACKUP_FRONTEND" ]; then
                    tar -xzf \$BACKUP_FRONTEND -C \${DEPLOY_PATH_FRONTEND}
                    sudo nginx -s reload
                fi
            '
        """
       }

    }
}
