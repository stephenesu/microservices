pipeline {
    agent any

    environment {
        DOCKER_HUB_USERNAME = 'stephenesu'
        SONAR_PROJECT_KEY = 'eazybank'
        K8S_NAMESPACE = 'eazybank'
        TRIVY_CACHE = '/var/jenkins_home/trivy-cache'
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

        stage('Build All Services') {
            parallel {
                stage('Build Accounts') {
                    steps { dir('section_14/accounts') { sh 'mvn clean package -DskipTests -B' } }
                }
                stage('Build Loans') {
                    steps { dir('section_14/loans') { sh 'mvn clean package -DskipTests -B' } }
                }
                stage('Build Cards') {
                    steps { dir('section_14/cards') { sh 'mvn clean package -DskipTests -B' } }
                }
                stage('Build Message') {
                    steps { dir('section_14/message') { sh 'mvn clean package -DskipTests -B' } }
                }
                stage('Build ConfigServer') {
                    steps { dir('section_14/configserver') { sh 'mvn clean package -DskipTests -B' } }
                }
                stage('Build EurekaServer') {
                    steps { dir('section_14/eurekaserver') { sh 'mvn clean package -DskipTests -B' } }
                }
                stage('Build GatewayServer') {
                    steps { dir('section_14/gatewayserver') { sh 'mvn clean package -DskipTests -B' } }
                }
            }
        }

        stage('Unit Tests') {
            steps {
                echo '========== Stage 3: Unit Tests =========='
                dir('section_14/accounts') {
                    sh 'mvn test -B'
                }
            }
            post {
                always {
                    dir('section_14/accounts') {
                        junit(allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml')
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo '========== Stage 4: SonarQube Analysis =========='
                dir('section_14/accounts') {
                    withSonarQubeEnv('sonarqube-server') {
                        sh 'mvn sonar:sonar -Dsonar.projectKey=eazybank -Dsonar.projectName=eazybank -B'
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

        stage('Trivy Scan') {
            parallel {
                stage('Scan Accounts') {
                    steps {
                        dir('section_14/accounts') {
                            sh """
                                /var/jenkins_home/bin/trivy fs \
                                    --exit-code 0 \
                                    --severity HIGH,CRITICAL \
                                    --format table \
                                    --timeout 10m \
                                    --scanners vuln \
                                    --cache-dir ${TRIVY_CACHE} \
                                    --skip-db-update \
                                    ./target
                            """
                        }
                    }
                }
                stage('Scan Loans') {
                    steps {
                        dir('section_14/loans') {
                            sh """
                                /var/jenkins_home/bin/trivy fs \
                                    --exit-code 0 \
                                    --severity HIGH,CRITICAL \
                                    --format table \
                                    --timeout 10m \
                                    --scanners vuln \
                                    --cache-dir ${TRIVY_CACHE} \
                                    --skip-db-update \
                                    ./target
                            """
                        }
                    }
                }
                stage('Scan Cards') {
                    steps {
                        dir('section_14/cards') {
                            sh """
                                /var/jenkins_home/bin/trivy fs \
                                    --exit-code 0 \
                                    --severity HIGH,CRITICAL \
                                    --format table \
                                    --timeout 10m \
                                    --scanners vuln \
                                    --cache-dir ${TRIVY_CACHE} \
                                    --skip-db-update \
                                    ./target
                            """
                        }
                    }
                }
            }
        }

        stage('Push All Images with Jib') {
            steps {
                echo '========== Stage 7: Push All Images to DockerHub =========='
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        for SERVICE in accounts loans cards message configserver eurekaserver gatewayserver; do
                            echo "Building and pushing $SERVICE..."
                            cd section_14/$SERVICE
                            mvn jib:build \
                                -Djib.to.auth.username=${DOCKER_USER} \
                                -Djib.to.auth.password=${DOCKER_PASS} \
                                -Djib.to.tags=${BUILD_NUMBER},latest \
                                -B
                            cd ../..
                        done
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo '========== Stage 8: Deploy to Kubernetes =========='
                withCredentials([file(
                    credentialsId: 'kubernetes-config',
                    variable: 'KUBECONFIG'
                )]) {
                    sh '''
                        export PATH=$PATH:/var/jenkins_home/bin

                        # Create namespace if it doesn't exist
                        kubectl get namespace eazybank || kubectl create namespace eazybank

                        # Deploy in order
                        kubectl apply -f section_14/kubernetes/kafka/ -n eazybank
                        kubectl apply -f section_14/kubernetes/configserver/ -n eazybank
                        kubectl apply -f section_14/kubernetes/eurekaserver/ -n eazybank
                        kubectl apply -f section_14/kubernetes/accounts/ -n eazybank
                        kubectl apply -f section_14/kubernetes/loans/ -n eazybank
                        kubectl apply -f section_14/kubernetes/cards/ -n eazybank
                        kubectl apply -f section_14/kubernetes/message/ -n eazybank
                        kubectl apply -f section_14/kubernetes/gatewayserver/ -n eazybank

                        # Wait for accounts to be ready
                        kubectl rollout status deployment/accounts -n eazybank --timeout=300s
                    '''
                }
            }
        }

    }

    post {
        success {
            echo '========================================='
            echo 'Pipeline SUCCEEDED! All services deployed.'
            echo "Namespace: eazybank"
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
