pipeline {
  agent any
  environment {
    PROJECT = 'wordsmith'
    IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_ID}"
    IMAGE = "nguyenphong8852/${PROJECT}:${IMAGE_TAG}"
  }
  stages {
    stage('Checkout') {
      steps {
        git branch: 'main',
            url: 'git@github.com:phongnt93/wordsmith.git',
            credentialsId: 'github-wordsmith-ssh'
      }
    }

    stage('Build & Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'jenkins-dockerhub', 
                                          usernameVariable: 'DOCKERHUB_USER',
                                          passwordVariable: 'DOCKERHUB_PWD')]) {
          sh '''
            echo "$DOCKERHUB_PWD" | docker login -u "$DOCKERHUB_USER" --password-stdin
            docker build -t $IMAGE .
            docker push $IMAGE
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        withCredentials([kubeconfigFile(credentialsId: 'orbstack-kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            kubectl apply -k $WORKSPACE
            kubectl rollout status deployment/wordsmith -n wordsmith
          '''
        }
      }
    }
  }
  post {
    success {
      echo "✅ Deployment finished: $IMAGE"
    }
    failure {
      echo "❌ CI/CD pipeline failed, check logs"
    }
  }
}

