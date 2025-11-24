pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "lgaratem"
        IMAGE_NAME     = "backend-test"

        DOCKERHUB_CRED_ID = "dockerhub-cred"
        GITHUB_CRED_ID    = "github-cred"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install & Test & Build') {
            agent {
                docker {
                    image 'node:18'
                    args '-u root:root'
                }
            }

            steps {
                sh 'node -v'
                sh 'npm -v'
                sh 'npm ci --prefer-offline --no-audit --progress=false'
                sh 'npm test'
                sh 'npm run build'
            }
        }

        stage('Docker Build (on Jenkins host)') {
            steps {
                script {
                    sh """
                        docker build -t ${DOCKERHUB_USER}/${IMAGE_NAME}:latest .
                        docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:latest ${DOCKERHUB_USER}/${IMAGE_NAME}:5
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([string(credentialsId: DOCKERHUB_CRED_ID, variable: 'DH_PASS')]) {
                    sh """
                        echo "$DH_PASS" | docker login -u ${DOCKERHUB_USER} --password-stdin
                        docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:latest
                        docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:5
                    """
                }
            }
        }
    }

    post {
        failure {
            echo "Pipeline falló. Revisa Console Output."
        }
    }
}
