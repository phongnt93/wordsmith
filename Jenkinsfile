pipeline {
  agent {
    kubernetes {
      label 'wordsmith-agent'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  dnsPolicy: Default
  hostAliases:
    - ip: "192.168.194.229"
      hostnames:
        - "harbor.local"
  imagePullSecrets:
    - name: harbor-pull
  volumes:
    - name: harbor-ca
      secret:
        secretName: harbor-ca-cert
    - name: kaniko-secret
      secret:
        secretName: kaniko-secret
  containers:
    - name: kaniko
      image: harbor.local/wordsmith/kaniko:latest
      command:
        - cat
      tty: true
      env:
        - name: SSL_CERT_DIR
          value: /kaniko/ca-cert
      volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker
        - name: harbor-ca
          mountPath: /kaniko/ca-cert
          readOnly: true
    - name: kubectl
      image: harbor.local/wordsmith/kubectl:latest
      command:
        - cat
      tty: true
      env:
        - name: SSL_CERT_DIR
          value: /etc/ssl/certs/harbor
      volumeMounts:
        - name: harbor-ca
          mountPath: /etc/ssl/certs/harbor
          readOnly: true
"""
    }
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Build & Push') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor \
              --context=$WORKSPACE \
              --dockerfile=Dockerfile \
              --destination=${HARBOR}/${PROJECT}/wordsmith-app:${BUILD_ID} \
              --cache=true
          '''
        }
      }
    }
    // … các stage còn lại …
  }
}

