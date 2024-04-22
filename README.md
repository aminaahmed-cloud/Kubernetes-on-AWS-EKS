# Deploying Applications on AWS EKS with Jenkins Pipeline

This guide provides a step-by-step walkthrough for deploying applications on Amazon Elastic Kubernetes Service (EKS) using Jenkins Pipeline. I'll cover setting up an EKS cluster, deploying applications, automating deployments with Jenkins, using Amazon Elastic Container Registry (ECR) as the Docker repository, and configuring autoscaling.

## Step 1: Create EKS Cluster

1. **Install eksctl**: Install eksctl, the official CLI tool for Amazon EKS, on your local machine.
2. **Create EKS cluster**: Use eksctl to create an EKS cluster with the desired configuration (e.g., 3 Nodes, 1 Fargate profile).
3. **Verify cluster creation**: Check the status of the EKS cluster by running `kubectl get nodes`.

## Step 2: Deploy MySQL and phpMyAdmin

1. **Deploy MySQL**: Deploy MySQL to the EKS cluster using Kubernetes manifests or Helm charts. Configure persistent storage and expose necessary ports.
2. **Deploy phpMyAdmin**: Deploy phpMyAdmin to the EKS cluster and configure it to connect to the MySQL database.
3. **Verify deployment**: Access phpMyAdmin through its service URL and ensure it can connect to MySQL.

## Step 3: Deploy Your Java Application

1. **Prepare Docker image**: Build a Docker image for your Java application and push it to a Docker registry.
2. **Deploy application**: Create Kubernetes deployment manifests or Helm charts for your Java application and deploy it to the EKS cluster.
3. **Scale application**: Optionally, scale the deployment to multiple replicas using `kubectl scale`.

## Setup Continuous Deployment with Jenkins

### Step 4: Automate Deployment

1. **Install Jenkins**: Install Jenkins on a server accessible to the EKS cluster.
2. **Install necessary plugins**: Install Jenkins plugins for Kubernetes integration and pipeline support.
3. **Create Jenkins pipeline**: Write a Jenkins pipeline script (Jenkinsfile) that automates the deployment process.
4. **Configure Jenkins credentials**: Add credentials for accessing Docker registry and Kubernetes cluster to Jenkins.
5. **Run pipeline**: Trigger the Jenkins pipeline and verify automated deployment.

### Step 5: Use ECR as Docker Repository

1. **Create ECR repository**: Create a repository in Amazon ECR to store Docker images.
2. **Tag and push Docker image**: Tag your Docker image with the ECR repository URI and push it to ECR.
3. **Update Jenkins pipeline**: Modify Jenkins pipeline script to use ECR as Docker repository for Java application image.
4. **Test pipeline**: Run Jenkins pipeline to build, tag, and push Docker image to ECR before deploying it to EKS.

### Step 6: Configure Autoscaling

1. **Enable Cluster Autoscaler**: Follow AWS documentation to enable Cluster Autoscaler for EKS cluster and configure minimum and maximum nodes.
2. **Deploy Horizontal Pod Autoscaler**: Configure Horizontal Pod Autoscaler (HPA) for Java application deployment to scale pods based on resource utilization.
3. **Test Autoscaling**: Deploy a load test or simulate increased traffic to trigger autoscaling. Monitor cluster to ensure it scales up and down based on configured policies.

