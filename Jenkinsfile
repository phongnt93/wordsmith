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
      image: gcr.io/kaniko-project/executor:debug
      command:
        - cat
      tty: true
      volumeMounts:
        - name: docker-credentials
          mountPath: /kaniko/.docker
        - name: workspace-volume
          mountPath: /workspace
    - name: kubectl
      image: bitnami/kubectl:1.27
      command:
        - sh
        - -c
        - sleep infinity
      tty: true
      volumeMounts:
        - name: workspace-volume
          mountPath: /workspace
  volumes:
    - name: docker-credentials
      projected:
        sources:
          - secret:
              name: dockerhub-secret
              items:
                - key: .dockerconfigjson
                  path: config.json
    - name: workspace-volume
      emptyDir: {}
"""
    }
  }

  environment {
    DOCKERHUB = 'docker.io'
    IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_ID}"
    IMAGE     = "${DOCKERHUB}/nguyenphong8852/wordsmith-app:${IMAGE_TAG}"
  }

  stages {
    stage('Checkout') {
      steps {
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
        container('kaniko') {
          sh '''
            /kaniko/executor \
              --context /workspace \
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
          sh '''
            kubectl apply -k /workspace/k8s
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

