pipeline {
  agent {
    kubernetes {
      label 'wordsmith-agent'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  # Ánh xạ hostnames về IP của node (Ingress Nginx NodePort)
  hostAliases:
  - ip: "192.168.194.229"
    hostnames:
    - "harbor.local"
  imagePullSecrets:
    - name: harbor-pull
  containers:
    - name: kaniko
      image: harbor.local:30649/wordsmith/kaniko:latest
      command: ['cat']
      tty: true
      volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker
    - name: kubectl
      image: harbor.local:30649/wordsmith/kubectl:latest
      command: ['cat']
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
      steps { checkout scm }
    }

    stage('Build & Push Image') {
      steps {
        container('kaniko') {
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
    success { echo "✅ Deployed ${IMAGE}" }
    failure { echo "❌ Pipeline failed, check logs." }
  }
}

