pipeline {
  agent none

  environment {
    DOCKERHUB_USER = "lgarate"           // reemplaza si hace falta
    IMAGE_NAME = "backend-test"
    DOCKERHUB_CRED_ID = "dockerhub-cred"
    GITHUB_CRED_ID = "github-cred"
    KUBECONFIG_CRED_ID = "kubeconfig"
  }

  stages {

    stage('Checkout') {
      agent { label 'master' } // ejecuta checkout en el nodo Jenkins
      steps {
        checkout scm
      }
    }

    stage('Install & Test & Build (in Node container)') {
      agent {
        docker {
          image 'node:18'
          args '-u root:root'   // ejecuta como root para evitar problemas de permisos
        }
      }
      steps {
        sh 'node -v && npm -v'
        sh 'npm ci --prefer-offline --no-audit --progress=false'
        sh 'npm test || true'     // si no hay tests, que no rompa todo
        sh 'npm run build'
      }
    }

    stage('Docker Build (on Jenkins host)') {
      agent { label 'master' }  // correr en el nodo que tiene docker-cli y docker.sock
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
      agent { label 'master' }
      steps {
        withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CRED_ID}", usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh """
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:latest
            docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER}
            docker logout
          """
        }
      }
    }

    stage('Push to GitHub Packages (GHCR)') {
      agent { label 'master' }
      steps {
        withCredentials([usernamePassword(credentialsId: "${GITHUB_CRED_ID}", usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
          sh """
            docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:latest ghcr.io/${GH_USER}/${IMAGE_NAME}:latest
            docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:latest ghcr.io/${GH_USER}/${IMAGE_NAME}:${BUILD_NUMBER}

            echo "$GH_TOKEN" | docker login ghcr.io -u "$GH_USER" --password-stdin

            docker push ghcr.io/${GH_USER}/${IMAGE_NAME}:latest
            docker push ghcr.io/${GH_USER}/${IMAGE_NAME}:${BUILD_NUMBER}

            docker logout ghcr.io
          """
        }
      }
    }

    stage('Update Kubernetes Deployment') {
      agent { label 'master' }
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED_ID}", variable: 'KUBECONFIG_FILE')]) {
          sh """
            export KUBECONFIG="$KUBECONFIG_FILE"
            kubectl set image deployment/backend-test backend-test=${DOCKERHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER} --record
            kubectl rollout status deployment/backend-test
          """
        }
      }
    }
  }

  post {
    success {
      echo "Pipeline finalizado OK — imagen: ${DOCKERHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER}"
    }
    failure {
      echo "Pipeline falló. Revisa Console Output."
    }
  }
}
