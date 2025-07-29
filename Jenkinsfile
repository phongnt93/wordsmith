pipeline {
  agent {
    kubernetes {
      label 'wordsmith-agent'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  imagePullSecrets:
    - name: dockerhub-secret
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      command:
        - cat
      tty: true
      volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker
    - name: kubectl
      image: bitnami/kubectl:1.27
      command:
        - cat
      tty: true
  volumes:
    - name: kaniko-secret
      secret:
        secretName: kaniko-secret
"""
    }
  }

  environment {
    DOCKERHUB = 'docker.io'
    IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_ID}"
    IMAGE = "${DOCKERHUB}/nguyenphong8852/wordsmith-app:${IMAGE_TAG}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout sshagent(credentials: ['github-wordsmith-ssh']) {
          sh 'git fetch && git checkout origin/main'
        }
      }
    }

    stage('Build & Push') {
      steps {
        container('kaniko') {
          sh """
            echo "{\"auths\": {\"${DOCKERHUB}\": {\"auth\": \"$(echo -n \"${DOCKERHUB_USER}:${DOCKERHUB_PWD}\" | base64)\"}}}" > /kaniko/.docker/config.json
            /kaniko/executor \
              --context=$WORKSPACE \
              --dockerfile=Dockerfile \
              --destination=${IMAGE} \
              --cache=true
          """
        }
      }
    }

    stage('Deploy to K8s') {
      steps {
        container('kubectl') {
          withCredentials([kubeconfigFile(credentialsId: 'orbstack-kubeconfig', variable: 'KUBECONFIG')]) {
            sh """
              kubectl apply -k $WORKSPACE
              kubectl rollout status deployment/wordsmith -n wordsmith
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployed ${IMAGE} successfully"
    }
    failure {
      echo "❌ Pipeline failed"
    }
  }
}
