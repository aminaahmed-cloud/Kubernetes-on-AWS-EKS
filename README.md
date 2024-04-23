# Deploying Applications on AWS EKS with Jenkins Pipeline

Right after you setup the cluster on LKE or Minikube and deployed your application inside, your manager comes to you to tell you that the company also wants to run Kubernetes on AWS. Again, with less overhead when managing just one platform. So they ask you to reconfigure your cluster on AWS and deploy your application there instead

## Step 1: Create EKS Cluster

1. **Install eksctl**: Install eksctl, the official CLI tool for Amazon EKS, on your local machine.
2. **Create EKS cluster**: Use eksctl to create an EKS cluster with the desired configuration (e.g., 3 Nodes, 1 Fargate profile).
3. **Verify cluster creation**: Check the status of the EKS cluster by running `kubectl get nodes`.

```sh
# create cluster with 3 EC2 instances and store access configuration to cluster in kubeconfig.my-cluster.yaml file 
eksctl create cluster --name=my-cluster --nodes=3 --kubeconfig=./kubeconfig.my-cluster.yaml

# create fargate profile in the cluster. It will apply for all K8s components in my-app namespace
eksctl create fargateprofile \
    --cluster my-cluster \
    --name my-fargate-profile \
    --namespace my-app

# point kubectl to your cluster - use absolute path to kubeconfigfile
export KUBECONFIG={absolute-path}/kubeconfig.my-cluster.yaml

# validate cluster is accessible and nodes and fargate profile created
kubectl get node
eksctl get fargateprofile --cluster my-cluster

```

<img src="https://i.imgur.com/ylkOhKD.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---

<img src="https://i.imgur.com/t7eabB9.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>



## Step 2: Deploy MySQL and phpMyAdmin

1. **Deploy MySQL**: Deploy MySQL to the EKS cluster using Kubernetes manifests or Helm charts. Configure persistent storage and expose necessary ports.
2. **Deploy phpMyAdmin**: Deploy phpMyAdmin to the EKS cluster and configure it to connect to the MySQL database.
3. **Verify deployment**: Access phpMyAdmin through its service URL and ensure it can connect to MySQL.

```

- Mysql Chart link: 
https://github.com/bitnami/charts/tree/master/bitnami/mysql 

```sh
# install Mysql chart 
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-release bitnami/mysql -f mysql-chart-values-eks.yaml

# deploy phpmyadmin with its configuration for Mysql DB access
kubectl apply -f db-config.yaml
kubectl apply -f db-secret.yaml
kubectl apply -f phpmyadmin.yaml

# access phpmyadmin and login to mysql db
kubectl port-forward svc/phpmyadmin-service 8081:8081

# access in browser on
localhost:8081

# login with one of these 2 credentials
"my-user" : "my-pass"
"root" : "secret-root-pass"

```


<img src="https://i.imgur.com/9OzFbFc.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---

<img src="https://i.imgur.com/sl3iJXa.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


---

<img src="https://i.imgur.com/HyRM8bM.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## Step 3: Deploy Your Java Application

1. **Prepare Docker image**: Build a Docker image for your Java application and push it to a Docker registry.
2. **Deploy application**: Create Kubernetes deployment manifests or Helm charts for your Java application and deploy it to the EKS cluster.
3. **Scale application**: Optionally, scale the deployment to multiple replicas using `kubectl scale`.

```sh

# Create namespace my-app to deploy our java application, because we are deploying java-app with fargate profile. And fargate profile we create applies for my-app namespace. 
kubectl create namespace my-app

# We now have to create all configuration and secrets for our java app in the my-app namespace

# Create my-registry-key secret to pull image 
DOCKER_REGISTRY_SERVER=docker.io
DOCKER_USER=your dockerID, same as for `docker login`
DOCKER_EMAIL=your dockerhub email, same as for `docker login`
DOCKER_PASSWORD=your dockerhub pwd, same as for `docker login`

kubectl create secret -n my-app docker-registry my-registry-key \
--docker-server=$DOCKER_REGISTRY_SERVER \
--docker-username=$DOCKER_USER \
--docker-password=$DOCKER_PASSWORD \
--docker-email=$DOCKER_EMAIL


# Again from k8s-deployment folder, execute following commands. By adding the my-app namespace, these components will be created with Fargate profile
kubectl apply -f db-secret.yaml -n my-app
kubectl apply -f db-config.yaml -n my-app
kubectl apply -f java-app.yaml -n my-app

```


## Setup Continuous Deployment with Jenkins

### Step 4: Automate Deployment

Now your application is running, and when you or others make changes to it, Jenkins pipeline builds the new image, but you have to manually deploy it into the cluster. From your experience you know how annoying that is for you and your team, so you want to automate deploying to the cluster as well.



1. **Install Jenkins**: Install Jenkins on a server accessible to the EKS cluster.
2. **Install necessary plugins**: Install Jenkins plugins for Kubernetes integration and pipeline support.
3. **Create Jenkins pipeline**: Write a Jenkins pipeline script (Jenkinsfile) that automates the deployment process.
4. **Configure Jenkins credentials**: Add credentials for accessing Docker registry and Kubernetes cluster to Jenkins.
5. **Run pipeline**: Trigger the Jenkins pipeline and verify automated deployment.

```sh
# Create an ECR registry for your java-app image

# Locally, on your computer: Create a docker registry secret for ECR
DOCKER_REGISTRY_SERVER=your ECR registry server - "your-aws-id.dkr.ecr.your-ecr-region.amazonaws.com"
DOCKER_USER=your dockerID, same as for `docker login` - "AWS"
DOCKER_PASSWORD=your dockerhub pwd, same as for `docker login` - get using: "aws ecr get-login-password --region {ecr-region}"

kubectl create secret -n my-app docker-registry my-ecr-registry-key \
--docker-server=$DOCKER_REGISTRY_SERVER \
--docker-username=$DOCKER_USER \
--docker-password=$DOCKER_PASSWORD

# SSH into server where Jenkins container is running
ssh -i {private-key-path} {user}@{public-ip}

# Enter Jenkins container
sudo docker exec -it {jenkins-container-id} -u 0 bash

# Install aws-cli inside Jenkins container
- Link: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install

# Install kubectl inside Jenkins container
- Link: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

apt-get update
apt-get install -y apt-transport-https ca-certificates curl
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubectl

# Install envsubst tool
- Link: https://command-not-found.com/envsubst

apt-get update
apt-get install -y gettext-base

# create 2 "secret-text" credentials for AWS access in Jenkins: 
- "jenkins_aws_access_key_id" for AWS_ACCESS_KEY_ID 
- "jenkins_aws_secret_access_key" for AWS_SECRET_ACCESS_KEY    

# Create 4 "secret-text" credentials for db-secret.yaml:
- id: "db_user", secret: "my-user"
- id: "db_pass", secret: "my-pass"
- id: "db_name", secret: "my-app-db"
- id: "db_root_pass", secret: "secret-root-pass"

```


<img src="https://i.imgur.com/GUaFhZE.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---

<img src="https://i.imgur.com/aghVJ3l.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---

<img src="https://i.imgur.com/T4DH7iS.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

### Step 5: Use ECR as Docker Repository

Now your application is running, and when you or others make changes to it, Jenkins pipeline builds the new image, but you have to manually deploy it into the cluster. From your experience you know how annoying that is for you and your team, so you want to automate deploying to the cluster as well.

So, the company wants to use ECR instead, again to have everything on 1 platform and also to let AWS manage the repository including storage, cleanups etc. Therefore you:

1. **Create ECR repository**: Create a repository in Amazon ECR to store Docker images.
2. **Tag and push Docker image**: Tag your Docker image with the ECR repository URI and push it to ECR.
3. **Update Jenkins pipeline**: Modify Jenkins pipeline script to use ECR as Docker repository for Java application image.
4. **Test pipeline**: Run Jenkins pipeline to build, tag, and push Docker image to ECR before deploying it to EKS.

```Jenkinsfile

#!/usr/bin/env groovy

pipeline {
    agent any
    environment {
        ECR_REPO_URL = '211125390603.dkr.ecr.eu-west-2.amazonaws.com/java-maven-app'
        IMAGE_REPO = "${ECR_REPO_URL}/java-app"
        IMAGE_NAME = "1.0-${BUILD_NUMBER}"
        CLUSTER_NAME = "my-cluster"
        CLUSTER_REGION = "eu-west-2"
        AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
    }
    stages {
        stage('build app') {
            steps {
               script {
                   echo "building the application..."
                   sh './gradlew clean build'
               }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    sh "docker build -t ${IMAGE_REPO}:${IMAGE_NAME} ."
                    sh "aws ecr get-login-password --region ${CLUSTER_REGION} | docker login --username AWS --password-stdin ${ECR_REPO_URL}"
                    sh "docker push ${IMAGE_REPO}:${IMAGE_NAME}"
                }
            }
        }
        stage('deploy') {
            environment {
                APP_NAME = 'java-app'
                APP_NAMESPACE = 'my-app'
                DB_USER_SECRET = credentials('db_user')
                DB_PASS_SECRET = credentials('db_pass')
                DB_NAME_SECRET = credentials('db_name')
                DB_ROOT_PASS_SECRET = credentials('db_root_pass')
            }
            steps {
                script {
                    sh "aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${CLUSTER_REGION}"

                    env.DB_USER = sh(script: 'echo -n $DB_USER_SECRET | base64', returnStdout: true).trim()
                    env.DB_PASS = sh(script: 'echo -n $DB_PASS_SECRET | base64', returnStdout: true).trim()
                    env.DB_NAME = sh(script: 'echo -n $DB_NAME_SECRET | base64', returnStdout: true).trim()
                    env.DB_ROOT_PASS = sh(script: 'echo -n $DB_ROOT_PASS_SECRET | base64', returnStdout: true).trim()
                    
                    echo 'deploying new release to EKS...'
                    sh 'envsubst < java-app-cicd.yaml | kubectl apply -f -'
                    sh 'envsubst < db-config-cicd.yaml | kubectl apply -f -'
                    sh 'envsubst < db-secret-cicd.yaml | kubectl apply -f -'
                }
            }
        }
    }
}
```

### Step 6: Configure Autoscaling

Now your application is running, whenever a change is made, it gets automatically deployed in the cluster etc. This is great, but you notice that most of the time the 3 nodes you have are underutilized, especially at the weekends, because your containers aren't using that many resources. However, your company is still paying full price for all of the servers.

You suggest to your manager that you will be able to save the company some infrastructure costs by configuring autoscaling. Your manager is happy with this suggestion and asks you to configure it. So you:


1. **Enable Cluster Autoscaler**: Follow AWS documentation to enable Cluster Autoscaler for EKS cluster and configure minimum and maximum nodes.
2. **Deploy Horizontal Pod Autoscaler**: Configure Horizontal Pod Autoscaler (HPA) for Java application deployment to scale pods based on resource utilization.
3. **Test Autoscaling**: Deploy a load test or simulate increased traffic to trigger autoscaling. Monitor cluster to ensure it scales up and down based on configured policies.

