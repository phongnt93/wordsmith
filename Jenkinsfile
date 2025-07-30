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
    - name: docker
      image: docker:24.0.5-dind
      securityContext:
        privileged: true
      command:
        - dockerd-entrypoint.sh
      args:
        - --host=tcp://0.0.0.0:2375
        - --host=unix:///var/run/docker.sock
      volumeMounts:
        - name: dockersock
          mountPath: /var/run/docker.sock
        - name: docker-config
          mountPath: /root/.docker
    - name: kubectl
      image: bitnami/kubectl:1.27
      command:
        - sh
        - -c
        - sleep infinity
      tty: true
      volumeMounts:
        - name: dockersock
          mountPath: /var/run/docker.sock
  volumes:
    - name: dockersock
      hostPath:
        path: /var/run/docker.sock
    - name: docker-config
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
    IMAGE_TAG    = "${env.BRANCH_NAME}-${env.BUILD_ID}"
    COMPOSE_FILE = "docker-compose.yml"
    // (nếu bạn dùng biến trong compose, export ở đây)
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

    stage('Build & Push with Docker Compose') {
      steps {
        container('docker') {
          sh """
            # Kiểm tra Docker daemon
            docker info
            # Build & tag theo docker-compose.yml
            docker compose -f ${COMPOSE_FILE} build
            # Push các image đã build (compose sẽ push theo image: trong file)
            docker compose -f ${COMPOSE_FILE} push
          """
        }
      }
    }

    stage('Deploy to K8s') {
      steps {
        container('kubectl') {
          sh """
            kubectl apply -k ${WORKSPACE}/k8s
            kubectl rollout status deployment/api -n wordsmith
            kubectl rollout status deployment/web -n wordsmith
            kubectl rollout status statefulset/db -n wordsmith || true
          """
        }
      }
    }
  }

  post {
    success {
      echo "✅ Build/push và Deploy lên K8s thành công!"
    }
    failure {
      echo "❌ Pipeline thất bại!"
    }
  }
}

