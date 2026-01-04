# Jenkins + Helm CI/CD Pipeline on Kubernetes
This project demonstrates how to set up a Jenkins CI/CD pipeline to deploy an application using Helm to a local Kubernetes cluster. It’s ideal for development or proof-of-concept setups where cloud resources are not required.

## Project Overview
We will be doing the following:

install and run Jenkins on a dedicated server

Install Helm on our dedicated jenkins server and Use Helm to deploy our apps on Amazon Eks (Note: We must have provisioned Amazon EKS cluster with nodes on our AWS)

Trigger deployments automatically via GitHub webhooks

Secure Kubernetes access using Kubeconfig

Deploy from a Helm chart structure

### Prerequisites

We must ensure the following:

AWS Account with CLI configured

EC2 Key pair and proper IAM roles for EKS and EC2

GitHub repository for our Helm chart and Jenkinsfile

Helm

Jenkins

## Project Setup

1. ✅Create an EKS Cluster using eksctl

Note: Before creating our cluster we need to set up our AWS CLI so we can manage aws from our terminal. Example of our CLI configuration is shown below

![AWS CLI](https://i.postimg.cc/fWmhXPkn/Screenshot-2025-07-19-070414.png)

Then we go ahead to create our cluster

```
eksctl create cluster \
  --name eksctl-helm-cluster \
  --version 1.29 \
  --region us-east-1 \
  --nodegroup-name jenkins-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```
![Cluster](https://i.postimg.cc/YSYJgGqw/Screenshot-2025-07-19-071030.png)

Run the following to confirm:

```
kubectl get nodes
```

You should see something like:

![cluster](https://i.postimg.cc/wBQwghjb/Screenshot-2025-07-19-113728.png)

2. 🚀 Provision Jenkins EC2 Server

Launch an EC2 instance (Ubuntu) with security group allowing:

Port 22 (SSH)

Port 8080 (Jenkins)

SSH into the instance and install Jenkins:

```
sudo apt update
sudo apt install openjdk-17-jdk -y
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt install jenkins -y
```
Start and enable Jenkins:

```
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

![Start Jenkins](https://i.postimg.cc/B6gnZmjj/Screenshot-2025-07-16-054500.png)

Access Jenkins at: http://<EC2-Public-IP>:8080

![Jenkins](https://i.postimg.cc/vmrKCjKP/Screenshot-2025-07-16-060119.png)




3. 🔐 Configure AWS Credentials on Jenkins Server

We must ensure AWS CLI is installed on Jenkins sever so they can access each other:

```
sudo apt install awscli -y
```
Then run:

```
aws configure
```
Now like step one, we must enter our Access Key, Secret Key, Region (e.g. us-east-1), and output format (e.g. json).


4. 📦 Install kubectl and Helm:

```
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# helm
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
5. ⚙️ Configure kubeconfig for Jenkins

Ensure kubeconfig is generated and available at:

```
mkdir -p /var/lib/jenkins/.kube
cp /home/ubuntu/.kube/config /var/lib/jenkins/.kube/config
chown -R jenkins:jenkins /var/lib/jenkins/.kube
chmod 600 /var/lib/jenkins/.kube/config
```

🧪 Application Deployment Using Helm

Our application should be packaged as a Helm chart. Example structure

```
webapp/
  ├── templates/
  │   ├── deployment.yaml
  │   ├── service.yaml
  ├── Chart.yaml
  └── values.yaml
Jenkinsfile
```

🧰 Jenkins Pipeline Setup
1. We create a Jenkins Pipeline Job

Set the pipeline source to Pipeline script from SCM

SCM: Git

Repo URL: https://github.com/<our-username>/Integrating-jenkins-with-Helm-.git

Script Path: Jenkinsfile

📝 Jenkinsfile

```
pipeline {
  agent any

  environment {
    HELM = '/usr/local/bin/helm'
    KUBECONFIG = '/var/lib/jenkins/.kube/config'
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

![Jenkinsfile](https://i.postimg.cc/ydd0H3dY/Screenshot-2026-01-04-074556.png)

## GitHub Webhook Integration (Auto-trigger Builds) 🔄 

Go to GitHub → Your Repo → Settings → Webhooks → Add Webhook

Payload URL: http://<your-local-ip>:8080/github-webhook/

Content Type: application/json

Events: Just the push event

Save

Make sure your Jenkins server is accessible from GitHub (use ngrok or localtunnel if needed):

![webhook](https://i.postimg.cc/RC3fQQdy/Screenshot-2025-07-20-001315.png)



## Verify the Deployment through Jenkins pipeline🔍
We are going to check if our pipeline trigger and run the content of our Jenkinsfile...

![jenkins trigger](https://i.postimg.cc/8k29spf6/Screenshot-2025-07-20-071729.png)

### Update Helm Chart and push changes

Now, we will open 'Values.yaml' in our webapp directory and change the replicacount to 3, then save.

![edit value.yaml](https://i.postimg.cc/prXYXmgD/Screenshot-2025-07-20-022548.png)

We are also going to open 'deployment.yaml' in 'template directory' under resources block, and update the resource requests as follows:

```
resources:

requests:

memory: "180Mi"

cpu: "120m"
```
![deployment.yaml](https://i.postimg.cc/fLHSCBr9/Screenshot-2025-07-21-050947.png)

If we check the image above, we will see our deployment.yaml is in modular form, referencing values.yaml, so we will need to edit our resource block from values.yaml instead.

![edit resource block](https://i.postimg.cc/d3b1NXmL/Screenshot-2025-07-21-051845.png)

We commit and push change to our GitHub repo so Jenkins can pick up our changes through webhook and apply our changes.

![commit and push](https://i.postimg.cc/tCJwBJYZ/Screenshot-2025-07-20-025741.png)

Now let's see if GitHub picks up our changes

![jenkins update](https://i.postimg.cc/TwhNFfRq/Screenshot-2025-07-20-072252.png)

Now we can see our Jenkins apply our changes, which means we have successfully integrated Helm with Jenkins
