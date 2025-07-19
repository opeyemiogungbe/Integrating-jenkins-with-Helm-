pipeline {
  agent any

  environment {
    HELM = '/usr/local/bin/helm'
    KUBECONFIG = '/home/ubuntu/.kube/config'  // use ubuntu's kubeconfig (NOT root)
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Verify Cluster Access') {
      steps {
        sh 'kubectl get nodes'
      }
    }

    stage('Helm Deploy') {
      steps {
        sh '${HELM} upgrade --install webapp ./webapp -f ./webapp/values.yaml --namespace default --create-namespace'
      }
    }
  }
}

