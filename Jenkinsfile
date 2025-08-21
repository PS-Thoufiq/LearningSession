pipeline {

    agent any

    tools {

        maven 'Maven-3.9.11'

        jdk 'JDK 21 '

    }

    environment {

        APP_NAME = "LearningSession"

        BETA_PORT = 8092

        GAMMA_PORT = 8093

        SERVER_IP = "localhost"

        LOG_DIR = "${WORKSPACE}/logs"

        DOCKER_IMAGE = "thoufiq_learning_session"

        DOCKER_TAG = "${env.BUILD_NUMBER}"

        BETA_URL = "http://${SERVER_IP}:${BETA_PORT}"

        GAMMA_URL = "http://${SERVER_IP}:${GAMMA_PORT}"

        PROD_URL = "http://${SERVER_IP}:${PROD_PORT}"

    }

    stages {

        stage('Checkout Main Repo') {

            steps {

                git url: 'https://github.com/puli-reddy/LearningSession.git', branch: 'main'

                sh 'chmod +x mvnw'

            }

        }

        stage('Checkout Integration Tests') {

            steps {

                dir('integration-tests') {

                    git url: 'https://github.com/puli-reddy/LearningSessionIntegrationTests.git', branch: 'main'

                    sh 'chmod +x mvnw'

                }

            }

        }

        stage('Create Log Directory') {

            steps {

                sh 'mkdir -p ${LOG_DIR}'

            }

        }

        stage('Build') {

            steps {

                sh './mvnw clean package -DskipTests'

            }

        }

        stage('Build Docker Image') {

            steps {

                script {

                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."

                }

            }

        }

        stage('Setup Network') {

            steps {

                sh 'docker network create spring-network || true'

            }

        }

        stage('Deploy to Beta') {

            steps {

                echo "Deploying to Beta environment on port ${BETA_PORT}"

                script {

                    sh "docker rm -f ${APP_NAME}-beta || true"

                    sh """

                        docker run -d --name ${APP_NAME}-beta --network spring-network -p ${BETA_PORT}:${BETA_PORT} \

                        -e SPRING_PROFILES_ACTIVE=beta \

                        ${DOCKER_IMAGE}:${DOCKER_TAG}

                    """

                    sleep(time: 30, unit: "SECONDS")

                    retry(3) {

                        sh "curl -f ${BETA_URL}/actuator/health || exit 1"

                    }

                    echo "Beta is running on ${BETA_URL}"

                }

            }

        }

        stage('Integration Tests on Beta') {

            steps {

                dir('integration-tests') {

                    sh """

                        ./mvnw verify -Dspring.profiles.active=beta -Dservice.url=${BETA_URL}

                        if [ -d "target/failsafe-reports" ]; then

                            grep -l "FAILURE" target/failsafe-reports/*.txt && exit 1 || exit 0

                        fi

                    """

                }

            }

        }

        stage('Deploy to Gamma') {

            steps {

                echo "Deploying to Gamma environment on port ${GAMMA_PORT}"

                script {

                    sh "docker rm -f ${APP_NAME}-gamma || true"

                    sh """

                        docker run -d --name ${APP_NAME}-gamma --network spring-network -p ${GAMMA_PORT}:${GAMMA_PORT} \

                        -e SPRING_PROFILES_ACTIVE=gamma \

                        ${DOCKER_IMAGE}:${DOCKER_TAG}

                    """

                    sleep(time: 30, unit: "SECONDS")

                    retry(3) {

                        sh "curl -f ${GAMMA_URL}/actuator/health || exit 1"

                    }

                    echo "Gamma is running on ${GAMMA_URL}"

                }

            }

        }

        stage('Approval for Prod') {

            steps {

                input message: 'Integration tests passed. Approve deployment to Prod?', ok: 'Deploy'

            }

        }

        stage('Deploy to Prod') {

            steps {

                echo "Deploying to Prod environment on port ${PROD_PORT}"

                script {

                    sh "docker rm -f ${APP_NAME}-prod || true"

                    sh """

                        docker run -d --name ${APP_NAME}-prod --network spring-network -p ${PROD_PORT}:${PROD_PORT} \

                        -e SPRING_PROFILES_ACTIVE=prod \

                        ${DOCKER_IMAGE}:${DOCKER_TAG}

                    """

                    sleep(time: 30, unit: "SECONDS")

                    retry(3) {

                        sh "curl -f ${PROD_URL}/actuator/health || exit 1"

                    }

                    echo "Prod is running on ${PROD_URL}"

                }

            }

        }

    }

    post {

        success {

            echo "Pipeline completed successfully!"

            echo "Beta: ${BETA_URL}"

            echo "Gamma: ${GAMMA_URL}"

            echo "Prod: ${PROD_URL}"

        }

        failure {

            echo "Pipeline failed. Check container logs."

            script {

                sh "docker logs ${APP_NAME}-beta || true"

                sh "docker logs ${APP_NAME}-gamma || true"

                sh "docker logs ${APP_NAME}-prod || true"

            }

        }

    

    }

}

