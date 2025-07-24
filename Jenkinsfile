pipeline {
  agent {
    kubernetes {
      defaultContainer 'kaniko'
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
      - name: kaniko-secret
        mountPath: /kaniko/.docker
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
      - cat
    tty: true
  volumes:
    - name: kaniko-secret
      secret:
        secretName: regcred
      """
    }
  }
  environment {
    // Định nghĩa registry Harbor và project
    HARBOR_URL = 'harbor.mycompany.com/library'
  }
  #triggers {
  #  // Kích hoạt pipeline khi có push lên nhánh main (có thể dùng GitHub hook)
  #  // triggerSimple('@daily') // ví dụ: hoặc githubPush()
  #}
  stages {
    stage('Build and Push Images') {
      steps {
        // Build image cho api và web bằng Kaniko
        container('kaniko') {
          sh """
            /kaniko/executor --context api --insecure \\
              --destination=${HARBOR_URL}/wordsmith-api:${env.BUILD_NUMBER}
            /kaniko/executor --context web --insecure \\
              --destination=${HARBOR_URL}/wordsmith-web:${env.BUILD_NUMBER}
          """
        }
      }
    }
    stage('Deploy to Kubernetes') {
      steps {
        // Triển khai bằng Kustomize
        container('kubectl') {
          withKubeConfig([credentialsId: 'kubeconfig-credential']) {
            sh 'kubectl apply -k ./k8s-manifests'
          }
        }
      }
    }
  }
}

