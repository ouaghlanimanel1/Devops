pipeline {
    agent any

    tools { 
        maven 'M2_HOME'
    }

    environment {
        DOCKER_IMAGE = "manel90162/devopspipline:latest"
        DOCKER_CREDENTIALS = "pipeline-exemple"
    }

    stages {
        stage('Checkout Git') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/ouaghlanimanel1/Devops.git'
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${DOCKER_IMAGE} -f docker/Dockerfile .'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: DOCKER_CREDENTIALS,
                    passwordVariable: 'DOCKER_PASS',
                    usernameVariable: 'DOCKER_USER'
                )]) {
                    script {
                        int retries = 3
                        for (int i = 1; i <= retries; i++) {
                            try {
                                // Properly separate commands to avoid shell errors
                                sh '''
                                    set -e
                                    export DOCKER_CLIENT_TIMEOUT=300
                                    export COMPOSE_HTTP_TIMEOUT=300
                                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                                    docker push ${DOCKER_IMAGE}
                                '''
                                echo "Docker push succeeded on attempt ${i}"
                                break
                            } catch (err) {
                                echo "Push attempt ${i} failed. Retrying..."
                                if (i == retries) { 
                                    error("Docker push failed after ${retries} attempts")
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        success { 
            echo 'Pipeline completed successfully!!' 
        }
        failure { 
            echo 'Pipeline failed. Check the logs!' 
        }
    }
}