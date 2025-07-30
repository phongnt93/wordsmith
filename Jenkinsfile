pipeline {
  agent {
    kubernetes {
      label 'wordsmith-agent'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      command: ["cat"]
      tty: true
      volumeMounts:
        - name: docker-credentials
          mountPath: /kaniko/.docker
    - name: kubectl
      image: bitnami/kubectl:1.27
      command: ["cat"]
      tty: true
  volumes:
    - name: docker-credentials
      projected:
        sources:
          - secret:
              name: dockerhub-secret
              items:
                - key: .dockerconfigjson
                  path: config.json
"""
    }
  }

  environment {
    DOCKERHUB    = 'docker.io'
    IMAGE_TAG    = "${env.BRANCH_NAME}-${env.BUILD_ID}"
    IMAGE        = "${DOCKERHUB}/nguyenphong8852/wordsmith-app:${IMAGE_TAG}"
  }

  stages {
    stage('Checkout') {
      steps {
        // Clone repo qua SSH
        checkout([$class: 'GitSCM',
                  branches: [[name: '*/main']],
                  userRemoteConfigs: [[
                    url: 'git@github.com:phongnt93/wordsmith.git',
                    credentialsId: 'github-wordsmith-ssh'
                  ]]])
      }
    }

    stage('Build & Push') {
      steps {
        // Chạy trong container kaniko, Kaniko sẽ tự đọc /kaniko/.docker/config.json
        container('kaniko') {
          sh '''
            /kaniko/executor \
              --context $WORKSPACE \
              --dockerfile Dockerfile \
              --destination ${IMAGE} \
              --cache=true
          '''
        }
      }
    }

    stage('Deploy to K8s') {
      steps {
        container('kubectl') {
          // Nếu kustomization.yaml nằm trong thư mục k8s/
          sh '''
            kubectl apply -k $WORKSPACE/k8s
            kubectl rollout status deployment/wordsmith -n wordsmith
          '''
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

