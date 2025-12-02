# Project_Amazon_Clone_Guide



# 🚀 Amazon Clone - Complete CI/CD Deployment Guide

![Amazon Clone Banner](./images/amazon-homepage.png)

A production-ready Amazon Clone application deployed using modern DevOps practices with Jenkins, Docker, SonarQube, Trivy, and AWS.

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Infrastructure Setup](#infrastructure-setup)
- [Jenkins Configuration](#jenkins-configuration)
- [Pipeline Stages](#pipeline-stages)
- [Deployment](#deployment)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)

---

## 🎯 Project Overview

This project demonstrates a complete CI/CD pipeline for deploying an Amazon Clone application with:

- **Infrastructure as Code**: Terraform for AWS resource provisioning
- **Continuous Integration**: Jenkins for automated builds and testing
- **Code Quality**: SonarQube for code analysis
- **Security Scanning**: Trivy and OWASP Dependency Check
- **Containerization**: Docker for application packaging
- **Deployment**: Automated deployment to AWS EC2

### Technology Stack

- **Frontend**: React.js / Node.js
- **CI/CD**: Jenkins 2.528.2
- **Code Quality**: SonarQube
- **Security**: Trivy, OWASP Dependency Check
- **Containerization**: Docker
- **Cloud Provider**: AWS (EC2, VPC, Security Groups)
- **IaC**: Terraform

---

## 🏗️ Architecture

```
Developer → GitHub → Jenkins → SonarQube → Docker Build → Docker Hub → AWS EC2
                         ↓
                    Trivy & OWASP
                    Security Scans
```

---

## ✅ Prerequisites

Before starting, ensure you have:

- AWS Account with appropriate permissions
- AWS CLI configured
- Terraform installed (v1.0+)
- Git installed
- DockerHub account
- Basic understanding of CI/CD concepts

---

## 🔧 Infrastructure Setup

### Step 1: Clone the Repository

```bash
mkdir amazon-devops
cd amazon-devops
git clone https://github.com/roshand33/Project-Amazon-Clone.git
cd Project-Amazon-Clone/JENKINS-TF
```

### Step 2: Configure AWS

```bash
aws configure
# Enter your AWS Access Key ID
# Enter your AWS Secret Access Key
# Enter your preferred region (e.g., ap-south-1)
```

### Step 3: Deploy Infrastructure with Terraform

![Terraform Apply](./images/terraform-apply.png)

```bash
# Initialize Terraform
terraform init

# Validate configuration
terraform validate

# Preview changes
terraform plan

# Apply configuration
terraform apply --auto-approve
```

**Resources Created:**
- 2 EC2 Instances (Jenkins Master & Monitoring)
- VPC and Subnets
- Security Groups with required ports
- IAM roles and policies

![EC2 Instances](./images/ec2-instances.png)

---

## ⚙️ Jenkins Configuration

### Step 1: Access Jenkins

After Terraform completes, access Jenkins at:

```
http://<EC2-PUBLIC-IP>:8080
```

![Jenkins Setup](./images/jenkins-setup.png)

Retrieve the initial admin password:

```bash
# SSH into Jenkins server
ssh -i your-key.pem ubuntu@<EC2-IP>

# Get initial password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Step 2: Install Required Plugins

Navigate to: **Manage Jenkins → Plugins → Available Plugins**

![Plugin Installation](./images/plugin-installation.png)

Install the following plugins:

1. ✅ Eclipse Temurin Installer
2. ✅ SonarQube Scanner
3. ✅ NodeJS Plugin
4. ✅ OWASP Dependency-Check
5. ✅ Docker (+ Docker Commons, Docker Pipeline, Docker API)
6. ✅ Pipeline Stage View

### Step 3: Configure Tools

Navigate to: **Manage Jenkins → Tools**

#### JDK Configuration
![JDK Configuration](./images/jdk-config.png)

- **Name**: `jdk17`
- **Install automatically**: ✅
- **Version**: jdk-17.0.10+7

#### Node.js Configuration
![NodeJS Configuration](./images/nodejs-config.png)

- **Name**: `node16`
- **Install automatically**: ✅
- **Version**: NodeJS 16.2.0

#### SonarQube Scanner Configuration
![SonarQube Scanner](./images/sonarqube-scanner-config.png)

- **Name**: `sonar-scanner`
- **Install automatically**: ✅
- **Version**: SonarQube Scanner 7.3.0.5189

#### OWASP Dependency-Check Configuration
![OWASP Config](./images/owasp-config.png)

- **Name**: `DP-Check`
- **Install automatically**: ✅
- **Version**: dependency-check 12.1.9

#### Docker Configuration
![Docker Config](./images/docker-config.png)

- **Name**: `docker`
- **Install automatically**: ✅

### Step 4: Configure Credentials

Navigate to: **Manage Jenkins → Credentials → System → Global credentials**

#### SonarQube Token

1. Access SonarQube: `http://<EC2-IP>:9000`
2. Login (default: admin/admin)
3. Navigate to: **My Account → Security → Generate Token**
4. Copy the token

![SonarQube Token](./images/sonarqube-token.png)

Add to Jenkins:
- **Kind**: Secret text
- **Secret**: `<your-sonarqube-token>`
- **ID**: `sonar-token`
- **Description**: SonarQube Authentication Token

#### DockerHub Credentials

- **Kind**: Username with password
- **Username**: `roshand22`
- **Password**: `<your-dockerhub-password>`
- **ID**: `docker`
- **Description**: DockerHub Credentials

### Step 5: Configure SonarQube Server

Navigate to: **Manage Jenkins → System → SonarQube servers**

![SonarQube Server Config](./images/sonarqube-server-config.png)

- **Name**: `sonar-server`
- **Server URL**: `http://13.127.228.37:9000`
- **Server authentication token**: `sonar-token`

---

## 🔄 Pipeline Stages

### Create New Pipeline

1. Click **New Item**
2. Enter name: `amazon`
3. Select **Pipeline**
4. Click **OK**

![Pipeline Creation](./images/pipeline-creation.png)

### Pipeline Script

```groovy
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/roshand33/Project-Amazon-Clone.git'
            }
        }
        stage("Sonarqube Analysis"){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Amazon \
                    -Dsonar.projectKey=Amazon '''
                }
            }
        }
        stage("quality gate"){
            steps {
                script {
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
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build -t amazon-clone ."
                       sh "docker tag amazon-clone roshand22/amazon-clone:latest"
                       sh "docker push roshand22/amazon-clone:latest"
                    }
                }
            }
        }
        stage("TRIVY Image Scan"){
            steps{
                sh "trivy image roshand22/amazon-clone:latest > trivyimage.txt"
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name amazon-clone -p 3000:3000 roshand22/amazon-clone:latest'
            }
        }
    }
}
```

### Pipeline Stage View

![Pipeline Stages](./images/pipeline-stages.png)

The pipeline consists of 10 stages:

1. **Declarative: Tool Install** - Sets up JDK and Node.js
2. **Clean Workspace** - Cleans previous build artifacts
3. **Checkout from Git** - Pulls latest code
4. **SonarQube Analysis** - Code quality analysis
5. **Quality Gate** - Validates code quality threshold
6. **Install Dependencies** - Installs npm packages
7. **OWASP FS SCAN** - Security vulnerability scan
8. **TRIVY FS SCAN** - Filesystem security scan
9. **Docker Build & Push** - Builds and pushes Docker image
10. **Deploy to Container** - Deploys application

---

## 🚀 Deployment

### Step 1: Run the Pipeline

Click **Build Now** to start the pipeline.

![Pipeline Running](./images/pipeline-running.png)

### Step 2: Monitor Build Progress

The build typically takes 15-20 minutes for the first run.

![Build Progress](./images/build-progress.png)

### Step 3: Verify Deployment

Once the pipeline succeeds, access your application:

```
http://<EC2-IP>:3000
```

![Application Running](./images/application-running.png)

### Successful Deployment

![Pipeline Success](./images/pipeline-success.png)

**Console Output:**
```bash
+ docker run -d --name amazon-clone -p 3000:3000 roshand22/amazon-clone:latest
72722362b44914a1b03771ee6b107c06c865581e35d25d22df3e6ec8958a21f09
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // stage
[Pipeline] End of Pipeline
Finished: SUCCESS
```

---

## 📊 Monitoring

### SonarQube Dashboard

Access SonarQube at: `http://<EC2-IP>:9000`

![SonarQube Dashboard](./images/sonarqube-dashboard.png)

View code quality metrics:
- Code coverage
- Code smells
- Vulnerabilities
- Security hotspots
- Duplications

### Jenkins Build History

Monitor all pipeline executions in the Jenkins dashboard.

![Build History](./images/build-history.png)

---

## 🔍 Troubleshooting

### Common Issues

#### 1. Quality Gate Failure

**Issue**: SonarQube quality gate fails the pipeline

**Solution**:
- Review SonarQube dashboard for issues
- Fix code quality problems
- Alternatively, configure quality gate settings in SonarQube

#### 2. Docker Permission Denied

**Issue**: Jenkins cannot access Docker

**Solution**:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

#### 3. Port Already in Use

**Issue**: Port 3000 already occupied

**Solution**:
```bash
# Stop existing container
docker stop amazon-clone
docker rm amazon-clone

# Or use different port
docker run -d --name amazon-clone -p 3001:3000 roshand22/amazon-clone:latest
```

#### 4. Trivy Not Installed

**Issue**: Trivy command not found

**Solution**:
```bash
# Install Trivy on Jenkins server
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

---

## 📁 Project Structure

```
Project-Amazon-Clone/
├── JENKINS-TF/
│   ├── main.tf              # EC2 instances configuration
│   ├── provider.tf          # AWS provider configuration
│   └── variables.tf         # Terraform variables
├── src/                     # Application source code
├── public/                  # Static assets
├── Dockerfile              # Docker configuration
├── package.json            # Node.js dependencies
└── README.md              # This file
```

---

## 🔐 Security Best Practices

1. **Never commit sensitive data** to the repository
2. **Use AWS Secrets Manager** for production credentials
3. **Enable MFA** on AWS account
4. **Regularly update** dependencies and Docker base images
5. **Review** SonarQube and Trivy reports
6. **Implement** OWASP security guidelines
7. **Use HTTPS** for production deployments

---

## 📝 Maintenance

### Update Pipeline

To modify the pipeline:
1. Navigate to the pipeline job
2. Click **Configure**
3. Update the pipeline script
4. Click **Save**
5. Run a new build

### Clean Up Resources

To destroy all infrastructure:

```bash
cd JENKINS-TF/
terraform destroy --auto-approve
```

### Docker Image Management

```bash
# View running containers
docker ps

# View all containers
docker ps -a

# Stop container
docker stop amazon-clone

# Remove container
docker rm amazon-clone

# Remove image
docker rmi roshand22/amazon-clone:latest

# Clean up unused resources
docker system prune -a
```

---

- GitHub: [@roshand33](https://github.com/roshand33)
- DockerHub: [roshand22](https://hub.docker.com/u/roshand22)
---

## 📞 Support

If you encounter any issues or have questions:

1. Check the [Troubleshooting](#troubleshooting) section
2. Review Jenkins console output for errors
3. Check SonarQube dashboard for code issues
4. Create an issue on GitHub
5. Refer to webhook configuration: [webhook_sonar_jenkins](https://github.com/roshand33/webhook_sonar_jenkins.git)

---

**⭐ If you find this project helpful, please give it a star!**

---

## 📸 Screenshots Gallery

### Application Screenshots

![Amazon Home](./images/amazon-home.png)
*Amazon Clone Home Page*

![Product Page](./images/product-page.png)
*Product Detail Page*

### Pipeline Screenshots

![Pipeline Overview](./images/pipeline-overview.png)
*Complete Pipeline Overview*

![Successful Build](./images/successful-build.png)
*Successful Pipeline Execution*

---
