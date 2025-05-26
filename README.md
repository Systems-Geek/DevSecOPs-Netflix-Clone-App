This project demonstrates how to deploy a Netflix clone web application on local machine running ubuntu server using modern DevSecOps tools. The setup includes Docker file for future containarization, static and dynamic security analysis using Trivy and Sonarqube, CI/CD with Jenkins, and observability with Prometheus and Grafana.
# DevSecOPs-Netflix-Clone-App
Netflix Clone Deployment with DevSecOps on Local Ubuntu Machine.

### ✅ Phase 1: Local Environment Setup

## 🔧 Step 1: Prepare Your Ubuntu Server (22.04)
Make sure your local Ubuntu server is up and running. SSH into it or access it directly if you're using it as a physical machine.

## 📂 Step 2: Clone Your Code Repository

Update the package list and clone your Netflix Clone application:

sudo apt update && sudo apt upgrade -y <br>
git clone https://github.com/Systems-Geek/DevSecOps-Netflix-Clone.git <br>
cd DevSecOps-Netflix-Clone <br>

Replace <your-username> with your GitHub handle.

## 🐳 Step 3: Install Docker and Launch the App
 
 # Install Docker and configure permissions:

sudo apt install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock

# Try building the container:


docker build -t netflix .
docker run -d --name netflix -p 8081:80 netflix:latest
The app may show an error due to a missing API key.

## 🔑 Step 4: Get and Add TMDB API Key
Sign up at TMDB.

Generate an API key under account settings.

Rebuild the Docker image with your TMDB API key:

docker build --build-arg TMDB_V3_API_KEY=<your_api_key> -t netflix .


### 🔐 Phase 2: Static Analysis and Image Scanning
📊 Install SonarQube


docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
Access at: http://localhost:9000
Login: admin / admin

# 🔍 Install Trivy for Image Scanning

sudo apt install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install trivy -y


Scan the Docker image:

trivy image netflix

### ⚙️ Phase 3: Jenkins CI/CD Pipeline
# 🧩 Install Jenkins on Local Machine

sudo apt update
sudo apt install fontconfig openjdk-17-jre -y

# Add Jenkins repo and key
wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins


sudo apt update
sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
Access Jenkins via: http://localhost:8080


# 🔧 Jenkins Configuration
Install the following plugins:

Eclipse Temurin Installer

SonarQube Scanner

NodeJs Plugin

OWASP Dependency-Check

Docker, Docker Pipeline, Docker Commons, Docker API

Add tools in Manage Jenkins → Global Tool Configuration:

JDK 17

Node.js 16

SonarQube scanner (add credentials/token)


# Add your DockerHub credentials under Manage Jenkins → Credentials (ID: docker).

# 🧪 Jenkins Pipeline Example (Declarative)



pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/<your-username>/DevSecOps-Netflix-Clone.git'
            }
        }
        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectKey=NetflixClone \
                    -Dsonar.projectName=NetflixClone
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'DP-Check'
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh 'docker build --build-arg TMDB_V3_API_KEY=<yourapikey> -t netflix .'
                        sh 'docker tag netflix <yourdockerhub>/netflix:latest'
                        sh 'docker push <yourdockerhub>/netflix:latest'
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image <yourdockerhub>/netflix:latest > trivyimage.txt'
            }
        }
        stage('Deploy Locally') {
            steps {
                sh 'docker run -d --name netflix -p 8081:80 <yourdockerhub>/netflix:latest'
            }
        }
    }
}


### 📈 Phase 4: Monitoring with Prometheus + Grafana

 # 🛠️ Install Prometheus

Download and configure:

wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
sudo mv prometheus-2.47.1.linux-amd64 /opt/prometheus


# Create Prometheus user and systemd service:


sudo useradd --no-create-home --shell /bin/false prometheus
sudo nano /etc/systemd/system/prometheus.service


# Paste and edit config as needed, then enable:

sudo systemctl daemon-reexec
sudo systemctl enable prometheus
sudo systemctl start prometheus

Prometheus: http://localhost:9090

## 🖥️ Install Node Exporter

Repeat similar steps to install node_exporter, then access metrics at http://localhost:9100.

## 📊 Install Grafana


wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana -y
sudo systemctl enable grafana-server
sudo systemctl start grafana-server

Access Grafana at: http://localhost:3000
Default: admin / admin

### 📢 Phase 5: Email Notifications (Optional)


Configure Jenkins to send emails by setting up:

SMTP settings in Manage Jenkins → Configure System

Email Extension Plugin

Add email recipients in your pipeline for alerts
