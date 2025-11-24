pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "lgaratem"
        IMAGE_NAME     = "backend-test"

        DOCKERHUB_CRED_ID = "dockerhub-cred"
        GITHUB_CRED_ID    = "github-cred"

        GITHUB_USER = "luis-garate"
        GHCR_IMAGE = "ghcr.io/${GITHUB_USER}/${IMAGE_NAME}"
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
                sh 'npm ci'
                sh 'npm test'
                sh 'npm run build'
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build -t ${DOCKERHUB_USER}/${IMAGE_NAME}:latest .
                    docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:latest ${DOCKERHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER}
                    docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:latest ${GHCR_IMAGE}:latest
                    docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:latest ${GHCR_IMAGE}:${BUILD_NUMBER}
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: DOCKERHUB_CRED_ID, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh """
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push ${DH_USER}/${IMAGE_NAME}:latest
                        docker push ${DH_USER}/${IMAGE_NAME}:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Push to GHCR') {
            steps {
                withCredentials([string(credentialsId: GITHUB_CRED_ID, variable: 'GH_PAT')]) {
                    sh """
                        echo "$GH_PAT" | docker login ghcr.io -u ${GITHUB_USER} --password-stdin
                        docker push ${GHCR_IMAGE}:latest
                        docker push ${GHCR_IMAGE}:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                        export KUBECONFIG=$KUBECONFIG_FILE

                        kubectl set image deployment/backend-deployment \
                            backend=${GHCR_IMAGE}:${BUILD_NUMBER} --record

                        kubectl rollout status deployment/backend-deployment
                    """
                }
            }
        }

    }

    post {
        failure {
            echo "❌ Pipeline falló. Revisar logs."
        }
        success {
            echo "✔ Deploy completado correctamente."
        }
    }
}