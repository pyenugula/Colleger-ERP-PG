pipeline {
    agent {
        label 'docker-agent'
    }

    environment {
        ACR_LOGIN_SERVER = "collegeerp.azurecr.io"
        WEB_IMAGE_NAME = "college-erp-web"
        POSTGRES_IMAGE_NAME = "college-erp-postgres"
        IMAGE_TAG = "${BUILD_NUMBER}"
        FULL_WEB_IMAGE = "${ACR_LOGIN_SERVER}/${WEB_IMAGE_NAME}:${IMAGE_TAG}"
        FULL_POSTGRES_IMAGE = "${ACR_LOGIN_SERVER}/${POSTGRES_IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {

        stage("Checkout") {
            steps {
                checkout scm
            }
        }

        stage("Build Web Docker Image") {
            steps {
                script {
                    // Build the web app image
                    sh """
                        docker build -t ${FULL_WEB_IMAGE} -f Dockerfile.web .
                    """
                } 
            }
        }

        stage("Build PostgreSQL Docker Image") {
            steps {
                script {
                    // Build the PostgreSQL image
                    sh """
                        docker build -t ${FULL_POSTGRES_IMAGE} -f Dockerfile.postgres .
                    """
                }
            }
        }

        stage("Trivy Image Scan (Web Image)") {
        steps {
        sh """
            trivy image \
                --severity HIGH,CRITICAL \
                --ignore-unfixed \
                --no-progress \
                ${FULL_WEB_IMAGE} || true
        """
    }
}

    stage("Trivy Image Scan (PostgreSQL Image)") {
    steps {
        sh """
            trivy image \
                --severity HIGH,CRITICAL \
                --ignore-unfixed \
                --exit-code 1 \
                --no-progress \
                ${FULL_POSTGRES_IMAGE} || true
        """
    }
}

    stage("Login to Azure Container Registry") {
    steps {
                withCredentials([usernamePassword(
                    credentialsId: 'acr-creds',
                    usernameVariable: 'ACR_USER',
                    passwordVariable: 'ACR_PASS'
                )]) {
                    sh """
                      echo "$ACR_PASS" | docker login \
                        ${ACR_LOGIN_SERVER} \
                        -u "$ACR_USER" \
                        --password-stdin
                    """
                }
            }
        }

        stage("Push Web Image to ACR") {
            steps {
                sh """
                  docker push ${FULL_WEB_IMAGE}
                """
            }
        }

        stage("Push PostgreSQL Image to ACR") {
            steps {
                sh """
                  docker push ${FULL_POSTGRES_IMAGE}
                """
            }
        }
    }

            // **Deploy Stage**: Deploy the images using Docker Compose
       stage("Deploy") {
            steps {
                script {
                    // Make sure the docker-compose.yml is present and configured
                    // Optionally, you can specify additional flags if needed
                    sh """
                        docker-compose -f docker-compose.yml up --build -d
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Web and PostgreSQL images built, scanned, and pushed successfully"
        }

        failure {
            echo "❌ Build failed — images were NOT pushed"
        }

        cleanup {
            sh """
              docker logout ${ACR_LOGIN_SERVER} || true
              docker image rm ${FULL_WEB_IMAGE} || true
              docker image rm ${FULL_POSTGRES_IMAGE} || true
              docker system prune -f || true
            """
        }
    }


