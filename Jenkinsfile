pipeline {
    agent any

    environment {
        DOCKERHUB = credentials('dockerhub-credentials')
        DOCKER_BUILDER_IP = credentials('docker-builder-ip')
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
        stage('Build and Push Docker Image') {
            steps{
                sshagent(['builder-server-key']){
                    sh """ ssh -o StrictHostKeyChecking=no azureuser@${DOCKER_BUILDER_IP} '
                        rm -rf ~/project &&
                        mkdir -p ~/project &&
                        git clone https://github.com/sudiptarathi2020/drnote.git ~/project  &&
                        cd ~/project &&
                        echo "${ENV_FILE}" > .env &&
                        echo \\${DOCKERHUB_PSW} | docker login -u \\${DOCKERHUB_USR} --password-stdin &&
                        docker build -t ${DOCKERHUB_USERNAME}/backend:${BUILD_NUMBER} -f backend/Dockerfile backend &&
                        docker push ${DOCKERHUB_USERNAME}/backend:${BUILD_NUMBER} &&
                        docker build -t ${DOCKERHUB_USERNAME}/frontend:${BUILD_NUMBER} -f frontend/Dockerfile frontend &&
                        docker push ${DOCKERHUB_USERNAME}/frontend:${BUILD_NUMBER} &&
                        docker system prune -af
                        '"""
                }
            }
        }
        stage('Deploy to Server') {
            steps {
                sshagent(['deployment-server-key']) {
                    sh """
                    scp -o StrictHostKeyChecking=no ${WORKSPACE}/drnote/docker-compose.yml azureuser@${DEPLOYMENT_SERVER}:~/drnote/docker-compose.yml &&
                    ssh -o StrictHostKeyChecking=no azureuser@${DEPLOYMENT_SERVER} '
                        set -x &&
                        mkdir -p ~/drnote &&
                        cd ~/drnote &&
                        echo "${ENV_FILE}" > .env &&
                        echo "BACKEND_IMAGE=${DOCKERHUB_USERNAME}/backend:${BUILD_NUMBER}" >> .env &&
                        echo "FRONTEND_IMAGE=${DOCKERHUB_USERNAME}/frontend:${BUILD_NUMBER}" >> .env &&
                        docker-compose pull &&
                        docker-compose down &&
                        ${params.RUN_MIGRATIONS ? 'docker-compose run --rm backend python manage.py migrate' : 'echo "Skipping migrations"'} &&
                        docker-compose up -d &&
                        docker system prune -af
                        '"""
                }
            }
        }
    }
}
