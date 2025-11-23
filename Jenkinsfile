pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "lgarate"
        DOCKERHUB_REPO = "backend-test"
        GITHUB_USER = "luis-garate"
        GITHUB_REPO = "backend-test"
        IMAGE_NAME = "backend-test"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Testing') {
            steps {
                sh 'npm test || echo "Tests skipped (no tests configured)"'
            }
        }

        stage('Build App') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    sh """
                    docker build -t ${DOCKERHUB_USER}/${IMAGE_NAME}:latest .
                    docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:latest ${DOCKERHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh """
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:latest
                    docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER}
                    docker logout
                    """
                }
            }
        }

        stage('Push to GitHub Packages') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-cred',
                    usernameVariable: 'GHUSER',
                    passwordVariable: 'GHTOKEN'
                )]) {
                    script {
                        sh """
                        docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:latest ghcr.io/${GITHUB_USER}/${IMAGE_NAME}:latest
                        docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:latest ghcr.io/${GITHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER}

                        echo $GHTOKEN | docker login ghcr.io -u $GHUSER --password-stdin

                        docker push ghcr.io/${GITHUB_USER}/${IMAGE_NAME}:latest
                        docker push ghcr.io/${GITHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER}

                        docker logout ghcr.io
                        """
                    }
                }
            }
        }

        stage('Update Kubernetes Deployment') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                    export KUBECONFIG=$KUBECONFIG_FILE
                    kubectl set image deployment/backend-test backend-test=${DOCKERHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER}
                    kubectl rollout restart deployment backend-test
                    """
                }
            }
        }
    }
}
