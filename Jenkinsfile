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
                sh """
                    docker build -t ${DOCKERHUB_USER}/${IMAGE_NAME}:latest .
                    docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:latest ${DOCKERHUB_USER}/${IMAGE_NAME}:5
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: DOCKERHUB_CRED_ID, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh """
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push ${DH_USER}/${IMAGE_NAME}:latest
                        docker push ${DH_USER}/${IMAGE_NAME}:5
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