pipeline {
    agent any

    tools {
        maven 'M2_HOME'
    }

    environment {
        DOCKER_CREDENTIALS = "pipeline-exemple"
        IMAGE_TAG = "${env.GIT_COMMIT}"

        NEXUS_HOST = "192.168.33.10:8083"
        NEXUS_REPO = "docker-repo"
        DOCKER_IMAGE = "${NEXUS_HOST}/${NEXUS_REPO}/student-app:${IMAGE_TAG}"

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
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_LOGIN')]) {
            withSonarQubeEnv('sonarqube') {
                sh """
                    mvn sonar:sonar \
                      -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                      -Dsonar.projectName="${SONAR_PROJECT_NAME}" \
                      -Dsonar.login=${SONAR_LOGIN} \
                      -Dsonar.java.binaries=target/classes
                """
            }
        }
    }
}


        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE} -f docker/Dockerfile .
                """
            }
        }

        stage('Push Docker Image to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: DOCKER_CREDENTIALS,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login ${NEXUS_HOST} -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}
                        docker logout ${NEXUS_HOST}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl --kubeconfig=${KUBECONFIG} get namespace ${K8S_NAMESPACE} \
                      || kubectl --kubeconfig=${KUBECONFIG} create namespace ${K8S_NAMESPACE}

                    kubectl --kubeconfig=${KUBECONFIG} apply -f kub/mysql-deployment.yaml -n ${K8S_NAMESPACE}
                    kubectl --kubeconfig=${KUBECONFIG} apply -f kub/spring-deployment.yaml -n ${K8S_NAMESPACE}

                    kubectl --kubeconfig=${KUBECONFIG} set image deployment/student-app \
                      student-app=${DOCKER_IMAGE} -n ${K8S_NAMESPACE}

                    kubectl --kubeconfig=${KUBECONFIG} rollout status deployment/student-app \
                      -n ${K8S_NAMESPACE} --timeout=120s

                    kubectl --kubeconfig=${KUBECONFIG} get pods -n ${K8S_NAMESPACE}
                """
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
