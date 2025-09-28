# 🚀 Deploy Netflix Clone on Cloud using Jenkins - DevSecOps Project!  

---

## **Phase 1: Initial Setup and Deployment**  

### 🖥️ Launch EC2 (Ubuntu 22.04)
- Provision an EC2 instance on AWS with **Ubuntu 22.04**.  
- Connect to the instance using **SSH**.  

### 📂 Clone the Code
- Update packages and clone the repository:  

```bash
sudo apt-get update
git clone https://github.com/AbhayShaw0125/Netflix-DevSecops-Project.git
```
## 🐳 Install Docker and Run the App

### Set up Docker on the EC2 instance:
```bash
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER  # Replace with your system's username, e.g., 'ubuntu'
newgrp docker
sudo chmod 777 /var/run/docker.sock
```
- Build and run your application:
```bash
docker build -t netflix .
docker run -d --name netflix -p 8081:80 netflix:latest

# To delete
docker stop <containerid>
docker rmi -f netflix
```
## ⚠️ Note: You will need a TMDB API key for the app to work.
### 🔑 Get the API Key

1. Go to the **[TMDB website](https://www.themoviedb.org/)**.
2. Log in or create an account.
3. Navigate to **Profile → Settings → API → Create**.
4. Fill in the details and submit. You will get your **API Key**.

- Rebuild the Docker image with your API key:
```bash
docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .
```
## Phase 2: Security
#### 🛡️ Install SonarQube & Trivy

###### SonarQube:
```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
- Access via http://<publicIP>:9000
- (Default username & password: admin)
  
###### Trivy
```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```
- Scan Docker images with Trivy:
  ```bash
  trivy image <imageid>
  ```
## Phase 3: CI/CD Setup
#### ⚙️ Install Jenkins
- First install jdk
  ```bash
  sudo apt update
  sudo apt install fontconfig openjdk-17-jre -y
  java -version
  ```
# Jenkins installation
```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt-get update
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
```
- Access Jenkins via: http://<publicIP>:8080
### 🔌 Install Jenkins Plugins

- **Eclipse Temurin Installer**  
- **SonarQube Scanner**  
- **NodeJS Plugin**  
- **Email Extension Plugin**  
- **OWASP Dependency-Check**  
- **Docker Plugins** (Docker, Docker Commons, Docker Pipeline, Docker API, docker-build-step)

### 🛠️ Configure Jenkins Tools

- **Global Tool Configuration** → Install **JDK 17** & **NodeJS 16**  
- **Add SonarQube token** in Jenkins Credentials  
- **Add DockerHub credentials** in Jenkins Credentials (ID: `docker`)

## 📦 Jenkins Pipeline Script
```groovy
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
        stage('🧹 Clean Workspace') {
            steps { cleanWs() }
        }
        stage('📂 Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/AbhayShaw0125/Netflix-DevSecops-Project.git'
            }
        }
        stage('🔍 SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage('✅ Quality Gate') {
            steps {
                script { waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' }
            }
        }
        stage('📦 Install Dependencies') {
            steps { sh "npm install" }
        }
        stage('🛡️ OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('🔎 TRIVY FS Scan') {
            steps { sh "trivy fs . > trivyfs.txt" }
        }
        stage('🐳 Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build --build-arg TMDB_V3_API_KEY=<yourapikey> -t netflix ."
                        sh "docker tag netflix abhayshaw0125/netflix:latest"
                        sh "docker push abhayshaw0125/netflix:latest"
                    }
                }
            }
        }
        stage('🔒 TRIVY Image Scan') {
            steps { sh "trivy image nasi101/netflix:latest > trivyimage.txt" }
        }
        stage('🚀 Deploy to Container') {
            steps { sh 'docker run -d --name netflix -p 8081:80 nasi101/netflix:latest' }
        }
    }
}
```
# 🛠️ Install Dependency-Check and Docker Tools in Jenkins

## 🔹 Install Dependency-Check Plugin

- Go to **Dashboard** in your Jenkins web interface.
- Navigate to **Manage Jenkins → Manage Plugins**.
- Click on the **Available** tab and search for **OWASP Dependency-Check**.
- Check the checkbox for **OWASP Dependency-Check** and click **Install without restart**.

## 🔹 Configure Dependency-Check Tool

- After installing the plugin, go to **Dashboard → Manage Jenkins → Global Tool Configuration**.
- Find the section for **OWASP Dependency-Check**.
- Add the tool's name, e.g., `DP-Check`.
- Save your settings.

## 🔹 Install Docker Tools and Docker Plugins

- Go to **Dashboard → Manage Jenkins → Manage Plugins → Available**.
- Search for **Docker** and install the following plugins:
  - Docker
  - Docker Commons
  - Docker Pipeline
  - Docker API
  - docker-build-step
- Click **Install without restart**.

## 🔹 Add DockerHub Credentials

- Go to **Dashboard → Manage Jenkins → Manage Credentials**.
- Click on **System → Global credentials (unrestricted)**.
- Click **Add Credentials**.
- Choose **Secret text** as the type of credential.
- Enter your **DockerHub credentials** (Username and Password).
- Give it an ID (e.g., `docker`) and click **OK**.

> ✅ Now, Dependency-Check and Docker plugins are installed, configured, and your DockerHub credentials are added.

---

# 📦 Jenkins Pipeline Script

```groovy
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('🧹 Clean Workspace'){
            steps{
                cleanWs()
            }
        }
        stage('📂 Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/AbhayShaw0125/Netflix-DevSecops-Project.git'
            }
        }
        stage('🔍 SonarQube Analysis'){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage('✅ Quality Gate'){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('📦 Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('🛡️ OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('🔎 TRIVY FS Scan') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage('🐳 Docker Build & Push'){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=<yourapikey> -t netflix ."
                       sh "docker tag netflix abhayshaw0125/netflix:latest"
                       sh "docker push abhayshaw0125/netflix:latest"
                    }
                }
            }
        }
        stage('🔒 TRIVY Image Scan'){
            steps{
                sh "trivy image abhayshaw0125/netflix:latest > trivyimage.txt"
            }
        }
        stage('🚀 Deploy to Container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 abhayshaw0125/netflix:latest'
            }
        }
    }
}
```
## ⚠️ If you get Docker login failed error

```bash
sudo su
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

**Phase 5: Notification**

. **Implement Notification Services:**
    - Set up email notifications in Jenkins or other notification mechanisms.
- This logs on any failure on the pipeline to the designated email

 Th```

