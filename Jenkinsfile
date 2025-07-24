pipeline {
  agent {
    kubernetes {
      label 'wordsmith-agent'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
  name: wordsmith-agent
spec:
  # Cho phép kéo image private từ Harbor
  imagePullSecrets:
    - name: harbor-pull
  containers:
    - name: kaniko
      image: harbor.local:30649/wordsmith/kaniko:latest
      command:
        - cat                       # Giữ container sống để Jenkins exec lệnh
      tty: true
      volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker
    - name: kubectl
      image: harbor.local:30649/wordsmith/kubectl:latest
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
    HARBOR    = 'harbor.local:30649'
    PROJECT   = 'wordsmith'
    IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_ID}"
    IMAGE     = "${HARBOR}/${PROJECT}/wordsmith-app:${IMAGE_TAG}"
  }

  stages {
    stage('Checkout') {
      steps {
        // Pull code từ GitHub
        checkout scm
      }
    }

    stage('Build & Push Image') {
      steps {
        container('kaniko') {
          // Build và push image qua Kaniko
          sh """
            /kaniko/executor \
              --context=\$WORKSPACE \
              --dockerfile=\$WORKSPACE/Dockerfile \
              --destination=${IMAGE} \
              --cache=true
          """
        }
      }
    }

    stage('Wait for Harbor Scan') {
      steps {
        // Chờ Harbor scan đảm bảo image sạch vulnerability ≥ Medium
        waitForHarborWebHook abortPipeline: true,
                           credentialsId: 'harbor-robot',
                           server: "${HARBOR}",
                           fullImageName: "${IMAGE}",
                           severity: 'Medium'
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          // Sử dụng kubeconfig đã lưu trong Jenkins để apply Kustomize
          withCredentials([kubeconfigFile(credentialsId: 'orbstack-kubeconfig', variable: 'KUBECONFIG')]) {
            sh """
              kubectl apply -k \$WORKSPACE
              kubectl rollout status deployment/wordsmith -n ${PROJECT}
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Pipeline succeeded: deployed image ${IMAGE}"
    }
    failure {
      echo "❌ Pipeline failed – xem logs phía trên để debug"
    }
  }
}

