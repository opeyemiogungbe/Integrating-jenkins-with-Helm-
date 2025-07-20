# Jenkins + Helm CI/CD Pipeline on Kubernetes
This project demonstrates how to set up a Jenkins CI/CD pipeline to deploy an application using Helm to a local Kubernetes cluster. It’s ideal for development or proof-of-concept setups where cloud resources are not required.

## Project Overview
We will be doing the following:

install and run Jenkins locally in a Docker container or natively

Install and Use Helm to deploy apps into Kubernetes on Docker Desktop

Trigger deployments automatically via GitHub webhooks

Securely configure Jenkins access to local Kubeconfig

Deploy from a Helm chart structure

### Prerequisites

We must ensure the following are installed on our local machine:

Docker Desktop

Kubernetes enabled in Docker Desktop

kubectl

Helm

Jenkins

GitHub repository for your Helm chart and Jenkinsfile


## Project Setup

1. ✅ We enable Kubernetes in Docker Desktop

```
Open Docker Desktop → Settings → Kubernetes → Enable Kubernetes.

Click "Apply & Restart".
```
Run the following to confirm:

```
kubectl get nodes
```

You should see something like:

```
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   ...   v1.xx.x
```

2. 🚀 Install Jenkins (Local or Docker)

Native Install (macOS/Linux)
```
brew install jenkins-lts         # For macOS (Homebrew)
sudo apt install jenkins         # For Ubuntu/Debian
```

Then start Jenkins:
```
sudo systemctl start jenkins
Access Jenkins at: http://localhost:8080
```
Option B: Run Jenkins in Docker
```
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v ~/.kube:/var/jenkins_home/.kube \
  -v /usr/local/bin/kubectl:/usr/local/bin/kubectl \
  -v /usr/local/bin/helm:/usr/local/bin/helm \
  jenkins/jenkins:lts
🔐 Make sure you bind-mount ~/.kube and helm, kubectl binaries into the container.
```
Using the first option, our Jenkins is successfully installed, and all the necessary plugins are installed.

![jenkins](https://i.postimg.cc/Xvp5XjBt/Screenshot-2025-07-16-060119.png)

3. 🔧 Setup kubectl and Helm

* kubectl
```
brew install kubectl               # macOS
sudo apt install kubectl -y       # Linux
```
* Helm
```
brew install helm
sudo snap install helm --classic
```

4. 🛡️ Grant Jenkins Kubernetes Access

Ensure Jenkins has access to your kubeconfig file. If using Docker, mount your local .kube/config file:

```
-v ~/.kube:/var/jenkins_home/.kube
```
Inside Jenkins or in the Jenkinsfile, set:
```
environment {
  KUBECONFIG = '/var/jenkins_home/.kube/config'
}
```
Or if Jenkins is installed natively, use:
```
environment {
  KUBECONFIG = "$HOME/.kube/config"
}
```
 ## Project Structure 📁

```
webapp/
  ├── templates/
  │   ├── deployment.yaml
  │   ├── service.yaml
  ├── Chart.yaml
  └── values.yaml
Jenkinsfile
```

## Jenkinsfile📝

```
pipeline {
  agent any

  environment {
    HELM = '/usr/local/bin/helm'
    KUBECONFIG = "$HOME/.kube/config"
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
```


## GitHub Webhook Integration (Auto-trigger Builds) 🔄 

Go to GitHub → Your Repo → Settings → Webhooks → Add Webhook

Payload URL: http://<your-local-ip>:8080/github-webhook/

Content Type: application/json

Events: Just the push event

Save

Make sure your Jenkins server is accessible from GitHub (use ngrok or localtunnel if needed):



## Verify the Deployment🔍
```
kubectl get pods
kubectl get svc
kubectl port-forward svc/webapp 8080:80
Visit: http://localhost:8080
```

💸 Resource Cleanup
Since this setup runs locally, there's no AWS billing risk. However, you can stop all Kubernetes pods with:
```
kubectl delete all --all
Or reset the cluster from Docker Desktop settings.
```
