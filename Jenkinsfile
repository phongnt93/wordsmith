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
      image: docker/compose:1.29.2-dind
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
          mountPath: /home/jenkins/.docker
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
    // IMAGE_TAG có thể dùng để override tag trong docker-compose nếu bạn cấu hình biến này trong compose
    IMAGE_TAG   = "${env.BRANCH_NAME}-${env.BUILD_ID}"
    COMPOSE_FILE = "docker-compose.yml"
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
            # đảm bảo đăng nhập Docker Hub
            docker info
            # build & tag theo docker-compose.yml
            docker-compose -f ${COMPOSE_FILE} build
            # push các image đã build (compose sẽ push theo image: trong file)
            docker-compose -f ${COMPOSE_FILE} push
          """
        }
      }
    }

    stage('Deploy to K8s') {
      steps {
        container('kubectl') {
          sh """
            # apply toàn bộ kustomization từ thư mục k8s/
            kubectl apply -k ${WORKSPACE}/k8s
            # chờ rollout hoàn thành
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

