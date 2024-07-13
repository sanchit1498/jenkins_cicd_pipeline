# DevSecOps Jenkins CI/CD Pipeline for a Node.js Application

## Overview
This project focuses on setting up a Jenkins CI/CD pipeline for a Node.js application, emphasizing DevSecOps practices. The pipeline incorporates stages for code quality analysis using SonarQube, security checks with OWASP tools, and Docker image scanning with Trivy. The objective is to automate the deployment process while upholding high standards of code quality and security.

![e43d72fe-ea98-482b-8d9e-8f6b81771ea3](https://github.com/user-attachments/assets/d16e7ad6-e570-40a4-b176-310b4264ab0b)

## Launch EC2 Instance
Create an AWS EC2 instance with the necessary permissions. Use `t2.large` as the instance type so that it can support the tools running for this project. Make note of the public IP address.

![601a8008-f99e-4fa9-870c-bb0aee4f7791](https://github.com/user-attachments/assets/76f6c746-e4b0-4e7d-95e4-2715a2ca7770)


## Jenkins Setup
Install Java:
```sh
sudo apt update
sudo apt install fontconfig openjdk-17-jre
```

Install Jenkins:
```sh
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins            
```

## Setting up Docker and Docker-compose
Install Docker and Docker-compose:
```sh
sudo apt-get install docker.io docker-compose
```
Add Ubuntu/Jenkins user to the Docker group:
```sh
sudo usermod -aG docker $USER
sudo usermod -aG docker jenkins
```

## SonarQube Server Setup
SonarQube serves as an autonomous code review tool designed to facilitate the delivery of Clean Code. Utilize Docker to deploy SonarQube:
```sh
docker run -itd --name sonarqube-server -p 9000:9000 sonarqube:lts-community
```

Access the SonarQube server via port 9000 and log in with the credentials user: admin, pass: admin.

## In Jenkins

+ Install the SonarQube Scanner plugin: Manage Jenkins > Plugins > Available Plugin > search for "SonarQube scanner" > install.

+ Add the token from SonarQube server into Jenkins: Manage Jenkins > Credentials > Add Credentials > Choose "Secret text" as kind, set ID as "sonar".
+ Add Docker Hub Credentials: Manage Jenkins > Credentials > Add Credentials > Choose "Username and Password" as kind, set ID as "DockerHubCreds".


## Establish the connection between the SonarQube server and Jenkins:

+ Navigate to Manage Jenkins.

+ Access the system settings.

+ Locate "SonarQube server".

+ Add a new SonarQube server configuration.

+ Save the changes.

![5ae10aa7-6613-4eed-a5ce-dae38597487e](https://github.com/user-attachments/assets/31dc1c69-0c0c-4d19-9394-bf21e995b415)


## Activate the SonarQube Scanner plugin:

+ Go to Manage Jenkins.
+ Choose Tools.
+ Look for "SonarQube Scanner installations".
+ Save your changes.

  ![472557f0-42bc-4d3c-b6c7-f411bbb0cb21](https://github.com/user-attachments/assets/4adebad1-f533-4257-908a-db4090226579)


## Set up a webhook on the SonarQube Server for Jenkins:

+ Navigate to Administration.
+ Select Configuration.
+ Locate the Webhooks section.
+ Click on Create to initiate the webhook setup process.

![8445721e-58d6-4877-a959-aaf593e1c4da](https://github.com/user-attachments/assets/d1423360-fceb-416d-8cb3-3cce22df9e59)


## Installing Trivy

Trivy is an open-source vulnerability scanner tailored for container images:

```sh
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

## Installing OWASP
Install OWASP dependency checker:

+ Add the OWASP dependency checker plugin: Manage Jenkins > Plugins > Available Plugins > search for "OWASP dependency checker" > install.
+ Enable OWASP: Manage Jenkins > Tools > find "Dependency-Check installations" > save your changes.

  ![8445721e-58d6-4877-a959-aaf593e1c4da](https://github.com/user-attachments/assets/d1423360-fceb-416d-8cb3-3cce22df9e59)


## Jenkins Pipeline Setup

This Jenkins pipeline automates the continuous integration and deployment process for a Node.js application. It comprises several stages:

+ Code: Clones the source code from a GitHub repository.
+ SonarQube Analysis: Conducts static code analysis using SonarQube to assess code quality.
+ SonarQube Quality Gates: Sets quality gates based on SonarQube analysis results.
+ OWASP: Utilizes OWASP dependency checker to scan for vulnerabilities in dependencies.
+ Build and Test: Builds a Docker image for the Node.js application.
+ Trivy: Conducts vulnerability scanning on the Docker image using Trivy.
+ Push to Private Docker Hub Repo: Pushes the Docker image to a private Docker Hub repository.
+ Deploy: Deploys the application using Docker Compose.
```sh
pipeline{
    agent any
    environment{
        SONAR_HOME = tool "Sonar"
    }
    stages{
        stage("code"){
            steps{
                git url:"https://github.com/sanchit1498/cicdnodejsApp" , branch:"master"
                echo "cloned success"
            }
        }
        stage("Sonarqube analysis"){
            steps{
              withSonarQubeEnv("Sonar"){
                 sh "$SONAR_HOME/bin/sonar-scanner -Dsonar.projectName=nodetodo -Dsonar.projectKey=nodetodo -X"
              }
            }
        }
        stage("SonarQube Quality Gates"){
            steps{
              timeout(time: 1, unit:"MINUTES"){
                waitForQualityGate abortPipeline: false
              }
            }
        }
        stage("OWASP"){
            steps{
               dependencyCheck additionalArguments: '--scan ./' , odcInstallation: 'OWASP'
               dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage("build and test"){
            steps{
                sh 'docker build -t node-app-batch-6:latest .'
               echo "build success"
            }
        }
        stage("Trivy"){
            steps{
              sh "trivy image node-app-batch-6"
            }
        }
        stage("Push to Private Docker Hub Repo"){
            steps{
                withCredentials([usernamePassword(credentialsId:"DockerHubCreds",passwordVariable:"dockerPass",usernameVariable:"dockerUser")]){
                sh "docker login -u ${env.dockerUser} -p ${env.dockerPass}"
                sh "docker tag node-app-batch-6:latest ${env.dockerUser}/node-app-batch-6:latest"
                sh "docker push ${env.dockerUser}/node-app-batch-6:latest"
                }
            }
        }
        stage("deploy"){
            steps{
               sh "docker compose down && docker compose up -d"
                 echo "deployed successfully"
            }
        }
    }
}
```
Now We Are Ready for Build
![7577d9cc-0979-4427-a41b-360a3318caa9](https://github.com/user-attachments/assets/c97e0f8e-6289-4183-a935-065e6d571217)


<img width="1512" alt="9d471f23-30bb-4b6c-89f8-4e4e3bc8cda9" src="https://github.com/user-attachments/assets/c4a42888-a63e-4b43-b112-dd409967e993">



