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
        string(name: 'DOCKER_TAG', defaultValue: 'latest', description: 'Docker image tag to deploy')
    }
    stages {
        stage('Checkout Code') {
            steps {
                git(url: 'https://github.com/sudiptarathi2020/drnote', branch: 'main')
            }
        }

        stage('Prepare .env File') {
            steps {
               // First, ensure workspace is writable
                sh "chmod -R 755 ${WORKSPACE}"
                
                // Create a deploy directory inside the workspace
                sh "mkdir -p ${WORKSPACE}/deploy_files"
                
                // Copy and prepare files with appropriate permissions
                withCredentials([
                    file(credentialsId: 'environment-file', variable: 'ENV_FILE_PATH'),
                    string(credentialsId: 'dockerhub-username', variable: 'DOCKER_USER')
                ]) {
                    script {
                        sh """
                        cp "\$ENV_FILE_PATH" "${WORKSPACE}/deploy_files/.env"
                        chmod 644 "${WORKSPACE}/deploy_files/.env"
                        """
                        
                        // Append Docker image info to env file without string interpolation
                        sh """
                        echo "BACKEND_IMAGE=\$DOCKER_USER/backend:${DOCKER_TAG}" >> "${WORKSPACE}/deploy_files/.env"
                        echo "FRONTEND_IMAGE=\$DOCKER_USER/frontend:${DOCKER_TAG}" >> "${WORKSPACE}/deploy_files/.env"
                        cp "${WORKSPACE}/docker-compose.yml" "${WORKSPACE}/deploy_files/"
                        cat "${WORKSPACE}/deploy_files/.env"
                        """
                    }
                }
            }
        }
        stage('Deploy to Server') {
            steps {
                sshagent(['deployment-server-key']) {
                    sh """
                    scp -o StrictHostKeyChecking=no "${WORKSPACE}/deploy_files/docker-compose.yml" "${WORKSPACE}/deploy_files/.env" azureuser@${DEPLOYMENT_SERVER}:~/drnote/ &&
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
        stage('Post-Deployment Cleanup') {
            steps {
                script {
                    // Clean up workspace after deployment
                    sh "rm -rf ${WORKSPACE}/deploy_files"
                }
            }
        }
    }
}
