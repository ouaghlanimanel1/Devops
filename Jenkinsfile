pipeline {
    agent any

    tools {
        maven 'M2_HOME'
    }

    environment {
        DOCKER_CREDENTIALS = "pipeline-exemple"
        IMAGE_TAG = "${env.BUILD_NUMBER}"  // Jenkins build number

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

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
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
                      # Login au registry Docker privé
                      echo "${DOCKER_PASS}" | docker login ${NEXUS_REGISTRY} -u "${DOCKER_USER}" --password-stdin
                      
                      # Push de l'image
                      docker push ${DOCKER_IMAGE}
                      
                      # Logout
                      docker logout ${NEXUS_REGISTRY}
                    """
                }
            }
        }

        stage('Create Kubernetes Docker Secret') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                      # Create or replace imagePullSecret
                      kubectl delete secret nexus-secret -n ${K8S_NAMESPACE} --ignore-not-found
                      kubectl create secret docker-registry nexus-secret \
                        --docker-server=${NEXUS_REGISTRY} \
                        --docker-username=${DOCKER_USER} \
                        --docker-password=${DOCKER_PASS} \
                        -n ${K8S_NAMESPACE}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                  # Ensure namespace exists
                  kubectl get namespace ${K8S_NAMESPACE} || kubectl create namespace ${K8S_NAMESPACE}

                  # Apply MySQL deployment (ensure revisionHistoryLimit)
                  kubectl apply -f kub/mysql-deployment.yaml -n ${K8S_NAMESPACE}
                  kubectl patch deployment mysql -n ${K8S_NAMESPACE} -p '{"spec":{"revisionHistoryLimit":1}}'

                  # Apply Spring deployment
                  kubectl apply -f kub/spring-deployment.yaml -n ${K8S_NAMESPACE}
                  kubectl patch deployment student-app -n ${K8S_NAMESPACE} -p '{"spec":{"revisionHistoryLimit":1}}'

                  # Add imagePullSecret to deployment
                  kubectl patch deployment student-app -n ${K8S_NAMESPACE} \
                    -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"nexus-secret"}]}}}}'

                  # Update deployment image
                  kubectl set image deployment/student-app \
                    student-app=${DOCKER_IMAGE} -n ${K8S_NAMESPACE}

                  # Restart deployments cleanly
                  kubectl rollout restart deployment/mysql -n ${K8S_NAMESPACE}
                  kubectl rollout restart deployment/student-app -n ${K8S_NAMESPACE}

                  # Wait for rollout to complete
                  kubectl rollout status deployment/mysql -n ${K8S_NAMESPACE} --timeout=300s
                  kubectl rollout status deployment/student-app -n ${K8S_NAMESPACE} --timeout=300s

                  # Show pods
                  kubectl get pods -n ${K8S_NAMESPACE}
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
