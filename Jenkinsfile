pipeline {
    agent any

    environment {
        DOCKER_HUB_USERNAME = 'stephenesu'
        SONAR_PROJECT_KEY = 'eazybank'
        SERVICE_NAME = 'accounts'
        SERVICE_DIR = 'section_14/accounts'
        IMAGE_NAME = "${DOCKER_HUB_USERNAME}/${SERVICE_NAME}"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    tools {
        maven 'maven3'
        jdk 'java21'
    }

    stages {

        stage('Checkout') {
            steps {
                echo '========== Stage 1: Checkout Code =========='
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo '========== Stage 2: Maven Build =========='
                dir("${SERVICE_DIR}") {
                    sh 'mvn clean compile -B'
                }
            }
        }

        stage('Unit Tests') {
            steps {
                echo '========== Stage 3: Unit Tests =========='
                dir("${SERVICE_DIR}") {
                    sh 'mvn test -B'
                }
            }
            post {
                always {
                    dir("${SERVICE_DIR}") {
                        junit(
                            allowEmptyResults: true,
                            testResults: '**/target/surefire-reports/*.xml'
                        )
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo '========== Stage 4: SonarQube Analysis =========='
                dir("${SERVICE_DIR}") {
                    withSonarQubeEnv('sonarqube-server') {
                        sh """
                            mvn sonar:sonar \
                                -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                -Dsonar.projectName=${SONAR_PROJECT_KEY} \
                                -B
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo '========== Stage 5: Quality Gate =========='
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package') {
            steps {
                echo '========== Stage 6: Package JAR =========='
                dir("${SERVICE_DIR}") {
                    sh 'mvn package -DskipTests -B'
                }
            }
        }

        stage('Build & Push Image with Jib') {
            steps {
                echo '========== Stage 7: Build & Push Docker Image with Jib =========='
                dir("${SERVICE_DIR}") {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            mvn jib:build \
                                -Djib.to.auth.username=${DOCKER_USER} \
                                -Djib.to.auth.password=${DOCKER_PASS} \
                                -Djib.to.tags=${IMAGE_TAG},latest \
                                -B
                        """
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                echo '========== Stage 8: Trivy Security Scan =========='
                sh """
                    /var/jenkins_home/bin/trivy image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

    }

    post {
        success {
            echo '========================================='
            echo 'Pipeline SUCCEEDED!'
            echo "Image pushed: ${IMAGE_NAME}:${IMAGE_TAG}"
            echo '========================================='
        }
        failure {
            echo '========================================='
            echo 'Pipeline FAILED!'
            echo '========================================='
        }
        always {
            echo 'Cleaning workspace...'
            cleanWs()
        }
    }
}
