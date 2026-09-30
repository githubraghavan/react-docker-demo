groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "raghavantaken98/react-docker-demo"
        DOCKER_CREDENTIALS = "dockerhub-credentials"
        APP_SERVER = "ubuntu@15.206.205.106"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Install Dependencies') {
            steps { sh 'npm ci' }
        }

        stage('Build React') {
            steps { sh 'npm run build' }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} -t ${DOCKER_IMAGE}:latest .
                """
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS}",
                    usernameVariable: 'raghavantaken98',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh """
                    docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    docker push ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Deploy to Application EC2') {
            steps {
                sh """
                    ssh -o StrictHostKeyChecking=no ${APP_SERVER} '
                        set -e

                        if ! command -v docker >/dev/null 2>&1; then
                            sudo apt update
                            sudo apt install -y ca-certificates curl
                            sudo install -m 0755 -d /etc/apt/keyrings
                            sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
                            sudo chmod a+r /etc/apt/keyrings/docker.asc
                            echo "deb [arch=\\$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \\$(. /etc/os-release && echo \\${UBUNTU_CODENAME:-\\$VERSION_CODENAME}) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
                            sudo apt update
                            sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
                        fi

                        sudo docker pull ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        sudo docker stop react-app || true
                        sudo docker rm react-app || true
                        sudo docker run -d --name react-app --restart unless-stopped -p 80:80 ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        sudo docker ps
                    '
                """
            }
        }
    }

    post {
        success { echo 'Build and deployment successful!' }
        failure { echo 'Build or deployment failed.' }
    }
}