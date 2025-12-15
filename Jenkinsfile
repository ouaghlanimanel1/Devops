pipeline {
    agent any

    tools {
        maven 'M2_HOME'
    }

    environment {
        DOCKER_CREDENTIALS = "pipeline-exemple"
        IMAGE_TAG = "${env.BUILD_NUMBER}"  // Use Jenkins build number to ensure unique tags

        NEXUS_HOST = "192.168.33.10:8081"
        NEXUS_REGISTRY = "192.168.33.10:8083"
        NEXUS_REPO = "docker-hosted"
        DOCKER_IMAGE = "${NEXUS_REGISTRY}/${NEXUS_REPO}/student-app:${IMAGE_TAG}"
        
        K8S_NAMESPACE = "devops"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"

        SONAR_PROJECT_KEY = "student-app"
        SONAR_PROJECT_NAME = "Student App"
    }

    stages {

        stage('Checkout Git') {
            steps {
                git branch: 'main', url: 'https://github.com/ouaghlanimanel1/Devops.git'
            }
        }

        stage('Build & Unit Tests') {
            steps {
                sh 'mvn clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                      mvn sonar:sonar \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName="${SONAR_PROJECT_NAME}"
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                  docker build -t ${DOCKER_IMAGE} -f docker/Dockerfile .
                '''
            }
        }

        stage('Push Docker Image to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                      # Login au registry Docker sur le port 8083
                      echo "${DOCKER_PASS}" | docker login ${NEXUS_REGISTRY} -u "${DOCKER_USER}" --password-stdin
                      
                      # Push de l'image
                      docker push ${DOCKER_IMAGE}
                      
                      # Logout
                      docker logout ${NEXUS_REGISTRY}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                  # Ensure namespace exists
                  kubectl --kubeconfig=${KUBECONFIG} get namespace ${K8S_NAMESPACE} \
                    || kubectl --kubeconfig=${KUBECONFIG} create namespace ${K8S_NAMESPACE}

                  # Deploy MySQL
                  kubectl --kubeconfig=${KUBECONFIG} apply -f kub/mysql-deployment.yaml -n ${K8S_NAMESPACE}


                  # Deploy Spring app
                  kubectl --kubeconfig=${KUBECONFIG} apply -f kub/spring-deployment.yaml -n ${K8S_NAMESPACE}

                  # Update image with unique tag
                  kubectl --kubeconfig=${KUBECONFIG} set image deployment/student-app \
                    student-app=${DOCKER_IMAGE} -n ${K8S_NAMESPACE}

                  # Optional: force delete stuck pods (safety)
                  kubectl --kubeconfig=${KUBECONFIG} delete pod -l app=student-app -n ${K8S_NAMESPACE} --force --grace-period=0 || true

                  # Wait for rollout
                  kubectl --kubeconfig=${KUBECONFIG} rollout status deployment/student-app \
                    -n ${K8S_NAMESPACE} --timeout=300s

                  # Show pods status
                  kubectl --kubeconfig=${KUBECONFIG} get pods -n ${K8S_NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
    }
}
