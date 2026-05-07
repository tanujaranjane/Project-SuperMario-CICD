# Project-SuperMario-CICD

---
### Create IAM role using following policies
````
AdministratorAccess   #only required
AmazonEKSClusterPolicy
AmazonEKSWorkerNodePolicy
AmazonEKS_CNI_Policy
AmazonEC2ContainerRegistryFullAccess
AmazonEC2FullAccess
AmazonVPCFullAccess
AutoScalingFullAccess
IAMFullAccess
CloudWatchFullAccess
````
### Launch Instance
  - AMI: Ubuntu
  - Instance Type: t3.medium

   ![instance](https://github.com/abhipraydhoble/Project-Super-Mario/assets/122669982/5fe51373-eaac-4f7c-9669-34c578277051)
   
  - Attach IAM to an Instance
![image](https://github.com/user-attachments/assets/c23f9d00-505d-4a0d-b07d-c6b21d419748)

![role-ec2](https://github.com/abhipraydhoble/Project-Super-Mario/assets/122669982/70cc0ebb-6063-4c4b-98df-7259a08749b8)

![modify-role](https://github.com/abhipraydhoble/Project-Super-Mario/assets/122669982/3e998e21-3654-43b0-8df0-496f009ef0a6)
---

### Connect to EC2-Instance
   ![connect-ec2](https://github.com/abhipraydhoble/Project-Super-Mario/assets/122669982/9d518e77-6f65-4153-acfc-790a6eaf669a)

---

### Script file to install Jenkins & Terraform (Give execute  permission and install)
````
#!/bin/bash
set -e

echo "🚀 Updating..."
sudo apt update -y && sudo apt install -y fontconfig openjdk-21-jre unzip curl snapd

# ---------------- Jenkins ----------------
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key | sudo tee /etc/apt/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update -y && sudo apt install -y jenkins
sudo systemctl enable jenkins && sudo systemctl start jenkins

# ---------------- Terraform ----------------
sudo snap install terraform --classic

# ---------------- kubectl ----------------
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl

# ---------------- AWS CLI ----------------
curl -s "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -q awscliv2.zip && sudo ./aws/install --update
rm -rf aws awscliv2.zip

````

---

### Install Required Plugins and Configure them in tools

````
AWS Credentials Plugin
Terraform 
Stage view 
````

### Add aws credentials in ManageJenkins-->Credentils


---
### Jenkins Pipeline
````
pipeline {
    agent any

    parameters {
        choice(
            name: 'ACTION',
            choices: ['apply', 'destroy'],
            description: 'Choose Terraform action'
        )
    }

    environment {
        AWS_DEFAULT_REGION = "ap-southeast-1"
        CLUSTER_NAME = "EKS_CLOUD"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
    }

    stages {

        stage('Clone Repo') {
            steps {
                git branch: 'main', url: 'https://github.com/abhipraydhoble/Project-SuperMario-CICD.git'
            }
        }

        stage('Terraform Init') {
            steps {
                dir('EKS-TF') {
                    sh 'terraform init'
                }
            }
        }

        stage('Terraform Apply') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                dir('EKS-TF') {
                    sh 'terraform apply -auto-approve'
                }
            }
        }

        stage('Configure kubeconfig') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                sh '''
                mkdir -p /var/lib/jenkins/.kube
                aws eks --region $AWS_DEFAULT_REGION \
                update-kubeconfig --name $CLUSTER_NAME \
                --kubeconfig $KUBECONFIG
                '''
            }
        }

        stage('Verify Cluster') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                sh '''
                export KUBECONFIG=$KUBECONFIG
                kubectl get nodes
                '''
            }
        }

        stage('Deploy to EKS') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                sh '''
                export KUBECONFIG=$KUBECONFIG
                kubectl apply -f deployment.yaml
                kubectl apply -f service.yaml
                '''
            }
        }

        stage('Terraform Destroy') {
            when {
                expression { params.ACTION == 'destroy' }
            }
            steps {
                input message: "Are you sure you want to DESTROY infrastructure?"
                dir('EKS-TF') {
                    sh 'terraform destroy -auto-approve'
                }
            }
        }
    }
}
````

### set permissions for cluster access
- Just change **NEW_USER_NAME** )iam user name) at the top to your new user, paste it into your terminal, and press Enter
````
NEW_USER="NEW_USER_NAME" && \
ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text) && \
aws eks create-access-entry --cluster-name EKS_CLOUD --region ap-southeast-1 --principal-arn arn:aws:iam::$ACCOUNT_ID:user/$NEW_USER --type STANDARD && \
aws eks associate-access-policy --cluster-name EKS_CLOUD --region ap-southeast-1 --principal-arn arn:aws:iam::$ACCOUNT_ID:user/$NEW_USER --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy --access-scope type=cluster
````

###  login into clutser
````
aws eks update-kubeconfig --name EKS_CLOUD --region ap-southeast-1
````

### access app
````
kubectl get nodes
kubectl get pods
kubectl get svc 
````
---

````
aws sts get-caller-identity
````

````
aws eks update-cluster-config \
    --name EKS_CLOUD \
    --access-config authenticationMode=API_AND_CONFIG_MAP
````
