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
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_CREDENTIALS = credentials('aws-creds')
        ECR_REPO = '897722705527.dkr.ecr.ap-south-1.amazonaws.com/my-project'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t $ECR_REPO:$IMAGE_TAG ."
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh """
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws ecr get-login-password --region $AWS_REGION \
                        | docker login --username AWS --password-stdin $ECR_REPO
                    """
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh "docker push $ECR_REPO:$IMAGE_TAG"
            }
        }

        stage('Deploy on Private EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                        ssh -i /home/ec2-user/test1.pem -o StrictHostKeyChecking=no ec2-user@10.0.2.128 '
                            aws ecr get-login-password --region $AWS_REGION \
                            | docker login --username AWS --password-stdin $ECR_REPO &&
                            docker rm -f nodeapp || true &&
                            docker pull $ECR_REPO:$IMAGE_TAG &&
                            docker run -d --name nodeapp -p 3000:3000 $ECR_REPO:$IMAGE_TAG
                        '
                    """
                }
            }
        }
    }
}
