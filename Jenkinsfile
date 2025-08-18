pipeline {
    agent any

    environment {
        DOCKERHUB = credentials('dockerhub-credentials')
        DEPLOYMENT_SERVER = credentials('deployment-server-ip')
        DOCKERHUB_USERNAME = credentials('dockerhub-username')
        ENV_FILE = credentials('environment-file')
    }

    parameters {
        booleanParam(name: 'RUN_MIGRATIONS', defaultValue: true, description: 'Run django database migrations')
    }
    stages {
        stage('Checkout Code') {
            steps {
                git(url: 'https://github.com/sudiptarathi2020/drnote', branch: 'main')
            }
        }
        stage('Prepare .env File') {
            steps {
                withCredentials([file(credentialsId: 'environment-file', variable: 'ENV_FILE_PATH')]) {
                    script {
                        def GIT_COMMIT = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
                        sh """
                        chmod -R u+w ${WORKSPACE} &&
                        cp \$ENV_FILE_PATH ${WORKSPACE}/.env &&
                        echo "BACKEND_IMAGE=${DOCKERHUB_USERNAME}/backend:${GIT_COMMIT}" >> ${WORKSPACE}/.env &&
                        echo "FRONTEND_IMAGE=${DOCKERHUB_USERNAME}/frontend:${GIT_COMMIT}" >> ${WORKSPACE}/.env &&
                        cat .env
                        """
                    }
                }
            }
        }
        stage('Deploy to Server') {
            steps {
                sshagent(['deployment-server-key']) {
                    sh """
                    scp -o StrictHostKeyChecking=no ${WORKSPACE}/docker-compose.yml .env azureuser@${DEPLOYMENT_SERVER}:~/drnote/ &&
                    ssh -o StrictHostKeyChecking=no azureuser@${DEPLOYMENT_SERVER} '
                        set -x &&
                        mkdir -p ~/drnote &&
                        cd ~/drnote &&
                        docker compose pull &&
                        docker compose down &&
                        ${params.RUN_MIGRATIONS ? 'docker compose run --rm backend python manage.py migrate' : 'echo "Skipping migrations"'} &&
                        docker compose up -d &&
                        docker system prune -af
                        '"""
                }
            }
        }
    }
}
