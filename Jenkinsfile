pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "raghavantaken98/react-docker-demo"
        DOCKER_CREDENTIALS = "dockerhub-credentials"
        APP_SERVER = "ubuntu@15.206.205.106"
        SSH_CREDENTIALS = "app-server-ssh-key"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "========================================="
                    echo "Node.js Version"
                    echo "========================================="
                    node -v

                    echo "========================================="
                    echo "NPM Version"
                    echo "========================================="
                    npm -v

                    echo "========================================="
                    echo "Installing Dependencies"
                    echo "========================================="

                    npm ci
                '''
            }
        }

        stage('Build React') {
            steps {
                sh '''
                    echo "========================================="
                    echo "Building React Application"
                    echo "========================================="

                    npm run build

                    echo "React build completed successfully."
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "========================================="
                    echo "Building Docker Image"
                    echo "========================================="

                    docker build \
                        -t "$DOCKER_IMAGE:$BUILD_NUMBER" \
                        -t "$DOCKER_IMAGE:latest" .

                    echo "Docker image built successfully."

                    docker images | grep react-docker-demo || true
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "========================================="
                        echo "Logging in to Docker Hub"
                        echo "========================================="

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        echo "Docker Hub login successful."
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    echo "========================================="
                    echo "Pushing Docker Image"
                    echo "========================================="

                    docker push "$DOCKER_IMAGE:$BUILD_NUMBER"

                    docker push "$DOCKER_IMAGE:latest"

                    echo "Docker images pushed successfully."
                '''
            }
        }

        stage('Test SSH Connection') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'app-server-ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USERNAME'
                    )
                ]) {

                    sh '''
                        echo "========================================="
                        echo "Testing SSH Connection"
                        echo "========================================="

                        echo "Application Server: $APP_SERVER"
                        echo "SSH Username: $SSH_USERNAME"

                        chmod 600 "$SSH_KEY"

                        ssh \
                            -i "$SSH_KEY" \
                            -o StrictHostKeyChecking=no \
                            -o UserKnownHostsFile=/dev/null \
                            "$SSH_USERNAME@15.206.205.106" \
                            "echo 'SSH connection successful'; hostname"

                        echo "========================================="
                        echo "SSH Authentication Successful"
                        echo "========================================="
                    '''
                }
            }
        }

        stage('Deploy to Application EC2') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'app-server-ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USERNAME'
                    )
                ]) {

                    sh '''
                        echo "========================================="
                        echo "Deploying to Application EC2"
                        echo "========================================="

                        echo "Application Server: $APP_SERVER"
                        echo "Docker Image: $DOCKER_IMAGE:$BUILD_NUMBER"

                        chmod 600 "$SSH_KEY"

                        ssh \
                            -i "$SSH_KEY" \
                            -o StrictHostKeyChecking=no \
                            -o UserKnownHostsFile=/dev/null \
                            "$SSH_USERNAME@15.206.205.106" \
                            "DOCKER_IMAGE='$DOCKER_IMAGE' BUILD_NUMBER='$BUILD_NUMBER' bash -s" <<'EOF'

set -e

echo "========================================="
echo "Connected to Application EC2"
echo "========================================="

echo "Hostname:"
hostname

echo "========================================="
echo "Checking Docker"
echo "========================================="

if ! command -v docker >/dev/null 2>&1; then

    echo "Docker not found."
    echo "Installing Docker..."

    sudo apt update

    sudo apt install -y \
        ca-certificates \
        curl

    sudo install -m 0755 -d /etc/apt/keyrings

    sudo curl -fsSL \
        https://download.docker.com/linux/ubuntu/gpg \
        -o /etc/apt/keyrings/docker.asc

    sudo chmod a+r /etc/apt/keyrings/docker.asc

    echo "Adding Docker repository..."

    echo "deb [arch=\$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \$(. /etc/os-release && echo \${UBUNTU_CODENAME:-\$VERSION_CODENAME}) stable" \
        | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt update

    sudo apt install -y \
        docker-ce \
        docker-ce-cli \
        containerd.io \
        docker-buildx-plugin \
        docker-compose-plugin

    sudo systemctl enable docker
    sudo systemctl start docker

else

    echo "Docker is already installed."

fi

echo "========================================="
echo "Docker Version"
echo "========================================="

sudo docker --version

echo "========================================="
echo "Docker Service Status"
echo "========================================="

sudo systemctl is-active docker || true

echo "========================================="
echo "Pulling Docker Image"
echo "========================================="

echo "Image: \$DOCKER_IMAGE:\$BUILD_NUMBER"

sudo docker pull "\$DOCKER_IMAGE:\$BUILD_NUMBER"

echo "========================================="
echo "Stopping Existing Container"
echo "========================================="

sudo docker stop react-app || true

echo "========================================="
echo "Removing Existing Container"
echo "========================================="

sudo docker rm react-app || true

echo "========================================="
echo "Starting New Container"
echo "========================================="

sudo docker run -d \
    --name react-app \
    --restart unless-stopped \
    -p 80:80 \
    "\$DOCKER_IMAGE:\$BUILD_NUMBER"

echo "========================================="
echo "Container Status"
echo "========================================="

sudo docker ps

echo "========================================="
echo "Container Logs"
echo "========================================="

sudo docker logs --tail 20 react-app

echo "========================================="
echo "Testing Application"
echo "========================================="

sleep 5

if curl -I http://localhost >/dev/null 2>&1; then

    echo "Application is responding on port 80."

else

    echo "Application did not respond on port 80."

    echo "========================================="
    echo "Container Logs"
    echo "========================================="

    sudo docker logs react-app

    exit 1

fi

echo "========================================="
echo "Deployment Completed Successfully"
echo "========================================="

EOF
                    '''
                }
            }
        }
    }

    post {

        success {
            echo '========================================='
            echo 'Build and Deployment Successful!'
            echo '========================================='
            echo "Application deployed to ${APP_SERVER}"
        }

        failure {
            echo '========================================='
            echo 'Build or Deployment Failed!'
            echo '========================================='
        }
    }
}

