# DevSecOps With Docker Scout Hotstar Clone

Certainly! Below is the complete blog post with detailed steps, including creating an EKS cluster:

---

# Deploying a Secure Hotstar Clone with DevSecOps CI/CD

In this comprehensive guide, we'll walk through the process of deploying a secure Hotstar Clone using DevSecOps principles. This project leverages AWS services, Docker, Jenkins, and various security tools to create an automated and secure deployment pipeline.

## Project Details

- **GitHub Repository:** [Hotstar Clone](https://github.com/RAM28EC/Hotstar-Clone.git)
- **Prerequisites:**
  - AWS account setup
  - Basic knowledge of AWS services
  - Understanding of DevSecOps principles
  - Familiarity with Docker, Jenkins, Java, SonarQube, AWS CLI, Kubectl, Terraform, and Docker Scout

## Step-by-Step Deployment Process

### Step 1A: Setting up AWS EC2 Instance and IAM Role

1. **Sign in to the AWS Management Console:**
   - Access the AWS Management Console using your credentials.

2. **Navigate to the EC2 Dashboard:**
   - Click on the "Services" menu at the top and select "EC2" under "Compute" in the left sidebar.

3. **Launch EC2 Instance:**
   - Click on the "Instances" link on the left sidebar.
   - Click the "Launch Instance" button.
   - Choose an Amazon Machine Image (AMI) - select an Ubuntu AMI.
   - Choose an Instance Type - select "t2.large."
   - Configure Instance Details, Add Storage, Add Tags, Configure Security Group, and Review.
   - Launch the instance and select an existing or create a new key pair.

4. **Access the EC2 Instance:**
   - Connect to the instance using SSH and the private key associated with the key pair.

### Step 1B: IAM Role

1. **Search for IAM in the AWS Console and click on Roles:**
   - Click on "Create Role."
   - Select the entity type as AWS service, use case as EC2, and click "Next."
   - Choose Administrator Access as the permission policy (for learning purposes) and complete the role creation.

2. **Attach IAM Role to EC2 Instance:**
   - In the EC2 Dashboard, select the instance.
   - Click on "Actions" → "Security" → "Modify IAM role."
   - Select the role created earlier and click "Update IAM role."

### Step 2: Installation of Required Tools on the Instance

#### Script 1: Install Java, Jenkins, Docker

```bash
sudo su  # Into root
vi script1.sh
```

```bash
# Script1 for Java, Jenkins, Docker

#!/bin/bash
sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins

# Install Docker
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker ubuntu
newgrp docker
```

#### Script 2: Install Terraform, kubectl, AWS CLI

```bash
vi script2

.sh
```

```bash
# Script 2 for Terraform, kubectl, AWS CLI

#!/bin/bash
# Install Terraform
sudo apt install wget -y
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Install Kubectl on Jenkins
sudo apt update
sudo apt install curl -y
curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt-get install unzip -y
unzip awscliv2.zip
sudo ./aws/install
```

### Run Scripts

```bash
# Give permissions to the scripts
sudo chmod 777 script1.sh
sudo chmod 777 script2.sh

# Run script1.sh
sh script1.sh

# Run script2.sh
sh script2.sh
```

### Step 3: Jenkins Job Configuration

#### Step 3A: EKS Provision Job

1. **Install Terraform Plugin in Jenkins:**
   - Go to Jenkins dashboard → Manage Jenkins → Plugins.
   - Search for "Terraform" and install the plugin.

2. **Add Terraform to Tools in Jenkins:**
   - Navigate to Manage Jenkins → Tools.
   - Add Terraform to the available tools.

3. **Create Jenkins Pipeline Job for EKS Provision:**
   - Create a new pipeline job in Jenkins.
   - Configure the pipeline script to provision an EKS cluster using Terraform.
     
  ```groovy 
  pipeline {
       agent any
       stages {
           stage('Checkout from Git') {
               steps {
                   git branch: 'main', url: 'https://github.com/RAM28EC/Hotstar-Clone.git'
               }
           }
           stage('Terraform version') {
               steps {
                   sh 'terraform --version'
               }
           }
           stage('Terraform init') {
               steps {
                   dir('EKS_TERRAFORM') {
                       sh 'terraform init'
                   }
               }
           }
           stage('Terraform validate') {
               steps {
                   dir('EKS_TERRAFORM') {
                       sh 'terraform validate'
                   }
               }
           }
           stage('Terraform plan') {
               steps {
                   dir('EKS_TERRAFORM') {
                       sh 'terraform plan'
                   }
               }
           }
           stage('Terraform apply') {
               steps {
                   dir('EKS_TERRAFORM') {
                       sh 'terraform ${action} --auto-approve'
                   }
               }
           }
       }
   }
   ```

4. **Run the Pipeline Job:**
   - Save and run the pipeline job in Jenkins.

5. **Verify EKS Cluster Creation:**
   - Check the AWS EKS console to verify the creation of the EKS cluster.

#### Step 3B: Hotstar Job

1. **Install Required Plugins in Jenkins:**
   - Go to Jenkins dashboard → Manage Jenkins → Plugins → Available Plugins.
   - Search and install the following plugins:
     - Eclipse Temurin installer
     - Sonarqube Scanner
     - NodeJs
     - Owasp Dependency-Check
     - Docker
     - Docker Commons
     - Docker Pipeline
     - Docker API
     - Docker-build-step

2. **Configure Global Tool Configuration in Jenkins:**
   - Navigate to Manage Jenkins → Tools → Install JDK(17) and NodeJs(16) → Click on Apply and Save.

3. **Install Kubernetes Plugin in Jenkins:**
   - Once installed, go to Manage Jenkins → Manage Credentials → Jenkins global → add credentials.

4. **Create Jenkins Pipeline Job for Deployment to Kubernetes:**
   - Create a new pipeline job in Jenkins.
   - Configure the pipeline script to deploy the application to a Kubernetes cluster.
```groovy 
 pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps {
                cleanWs()
            }
        }
        stage('checkout form Git'){
            steps{
                git branch: 'main', url: 'https://github.com/RAM28EC/Hotstar-Clone.git'

            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Hotstar \
                    -Dsonar.projectKey=Hotstar'''
                }
            }
        }
        stage('quality gate'){
            steps{
                script{
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Docker Scout FS') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh 'docker-scout quickview fs://.'
                       sh 'docker-scout cves fs://.'
                   }
                }   
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "docker build -t hotstar ."
                       sh "docker tag hotstar 28cloud/hotstar:latest "
                       sh "docker push 28cloud/hotstar:latest"
                    }
                }
            }
        }
        stage('Docker Scout Image') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh 'docker-scout quickview 28cloud/hotstar:latest'
                       sh 'docker-scout cves 28cloud/hotstar:latest'
                       sh 'docker-scout recommendations 28cloud/hotstar:latest'
                   }
                }   
            }
        }
        stage("deploy_docker"){
            steps{
                sh "docker run -d --name hotstar -p 3000:3000 28cloud/hotstar:latest"
            }
        }
        stage('Deploy to kubernets'){
            steps{
                script{
                    dir('K8S') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                        }
                    }
                }
            }
        }

    }
}
```
5. **Run the Pipeline Job:**
   - Save and run the pipeline job in Jenkins.

6. **Check Kubernetes Deployment:**
   - After a successful run, check the Kubernetes cluster for the deployment.

### Step 4: Clean-Up Process

#### Destroy EKS Cluster

1. **Run Jenkins Job for Terraform-EKS:**
   - Go to Jenkins Dashboard.
   - Click on the Terraform-EKS job.
   - Build with parameters and select the "destroy" action.

2. **Wait for Cluster Deletion:**
   - Wait for approximately 10 minutes for the EKS cluster to be deleted.

3. **Delete EC2 Instance and IAM Role:**
   - Remove the EC2 instance and IAM role created for the project.

4. **Check Load Balancer:**
   - Verify if the load balancer associated with the EKS cluster is deleted.

## Conclusion

By following this step-by-step guide, you have successfully deployed a secure Hotstar Clone using DevSecOps CI/CD practices on AWS. The detailed integration of security measures throughout the deployment pipeline showcases the power of DevSecOps methodologies in ensuring efficiency and robust security against potential threats.

### Key Highlights

[...]

---

Feel free to customize this blog post further based on your specific requirements and preferences.
