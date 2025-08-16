# 🚀 AWS Secure Application Deployment

This project demonstrates a **secure and automated CI/CD pipeline** for deploying containerized applications into a **private EC2 instance** within a VPC, using **Jenkins, Docker, Amazon ECR, and Bastion Host** for secure access.

---

## 📌 Features
- ✅ **End-to-End Deployment Pipeline** using Jenkins.
- ✅ **Dockerized Application Build** and push to **Amazon Elastic Container Registry (ECR)**.
- ✅ **Bastion Host (Jump Server)** setup for secure SSH access to **Private EC2**.
- ✅ **Private EC2 Deployment** with Docker runtime.
- ✅ **Secure AWS IAM Roles & Policies** (no hardcoded credentials).
- ✅ **Scalable & Production-Ready** architecture.

---

## 🏗️ Architecture

Developer → GitHub → Jenkins → Docker Build → ECR → Bastion Host → Private EC2 (App Running)


### Components
- **GitHub**: Hosts application source code and Jenkinsfile.
- **Jenkins**: CI/CD automation server for build, push, and deployment.
- **Docker**: Containerizes the application.
- **Amazon ECR**: Stores and manages Docker images.
- **Bastion Host**: Public EC2 instance for secure SSH tunneling.
- **Private EC2**: Runs the application inside Docker, isolated in private subnet.
- **IAM Roles**: Used for secure authentication (no hardcoded keys).

---

## ⚙️ Prerequisites
- AWS account with admin access.
- VPC setup with **public** and **private** subnets.
- Security groups:
  - Bastion Host: SSH allowed from Jenkins/your IP.
  - Private EC2: SSH only from Bastion, app port (3000) exposed as needed.
- Jenkins installed (with Docker, AWS CLI, SSH Agent Plugin).
- IAM roles:
  - Jenkins IAM User → ECR push permissions.
  - Private EC2 IAM Role → ECR pull permissions.

---

## 🚦 CI/CD Pipeline Flow
1. **Code Checkout** from GitHub.
2. **Build Docker Image** for the app.
3. **Authenticate & Push** image to ECR.
4. **SSH (via Bastion)** into private EC2.
5. **Pull latest image** from ECR.
6. **Run/Restart container** on private EC2.

---

## 📜 Jenkinsfile (Simplified Example)
```groovy
pipeline {
  agent any
  environment {
    AWS_REGION    = 'ap-south-1'
    ECR_REGISTRY  = '<account>.dkr.ecr.ap-south-1.amazonaws.com'
    ECR_REPO_NAME = 'my-project'
    ECR_REPO      = "${ECR_REGISTRY}/${ECR_REPO_NAME}"
    IMAGE_TAG     = "${BUILD_NUMBER}"
    BASTION_HOST  = "<PUBLIC_EC2_IP>"
    PRIVATE_HOST  = "10.0.2.128"
  }
  stages {
    stage('Build & Push') {
      steps {
        sh '''
          docker build -t ${ECR_REPO}:${IMAGE_TAG} .
          aws ecr get-login-password --region ${AWS_REGION} \
            | docker login --username AWS --password-stdin ${ECR_REGISTRY}
          docker push ${ECR_REPO}:${IMAGE_TAG}
        '''
      }
    }
    stage('Deploy') {
      steps {
        sshagent(['bastion-ssh-key']) {
          sh '''
            ssh -A ec2-user@${BASTION_HOST} ssh ec2-user@${PRIVATE_HOST} '
              aws ecr get-login-password --region ${AWS_REGION} \
                | docker login --username AWS --password-stdin ${ECR_REGISTRY}
              docker rm -f nodeapp || true
              docker pull ${ECR_REPO}:${IMAGE_TAG}
              docker run -d --name nodeapp -p 3000:3000 ${ECR_REPO}:${IMAGE_TAG}
            '
          '''
        }
      }
    }
  }
}
