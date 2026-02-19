<img width="1400" height="735" alt="image" src="https://github.com/user-attachments/assets/cefaf321-aef8-47c7-b9b1-53b33d432db3" />

Docker Project: 

CI/CD Pipeline with Jenkins, SonarQube, and Trivy 
Project Overview

This project demonstrates the implementation of a complete CI/CD pipeline using Jenkins, Docker, SonarQube, Trivy, and other DevSecOps tools. The pipeline builds, scans, and deploys a sample application (Zomato Project) in a Docker container.


Step 1: Launch an EC2 Instance

•	Instance Type: t2.large

•	Operating System: Linux (Amazon Linux 2 / CentOS)

•	Security Groups:

•	8080 for Jenkins

•	9000 for SonarQube

•	3000 for application deployment


Step 2: Install Required Tools
Jenkins Installation
sudo yum update -y

sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/ jenkins.repo

sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key sudo yum upgrade -y

sudo yum install java-17 -y sudo yum install jenkins -y sudo systemctl enable jenkins sudo systemctl start jenkins

<img width="975" height="511" alt="image" src="https://github.com/user-attachments/assets/24d6050e-84e7-4074-9443-49d86409ec5e" />

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/9ad75ead-2eb3-4e0c-a861-e2fa2ce5b450" />

Git Installation

sudo yum install git -y

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/3a29a66c-5449-4dbf-b093-40f44f2d8e82" />

Docker Installation

sudo yum install docker -y 

sudo systemctl enable

docker sudo systemctl start docker

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/d26cad0a-312c-4ad5-9ae7-c7f7ff69440b" />

Trivy Installation

wget https://github.com/aquasecurity/trivy/releases/download/v0.18.3/ trivy_0.18.3_Linux-64bit.tar.gz

tar zxvf trivy_0.18.3_Linux-64bit.tar.gz sudo mv trivy /usr/local/bin/

echo 'export PATH=$PATH:/usr/local/bin/' >> ~/.bashrc 

source ~/.bashrc

<img width="975" height="530" alt="image" src="https://github.com/user-attachments/assets/0a0b3a7f-1eb9-44f4-ac17-66c8ff1e1d19" />

Step 3: Install Jenkins Plugins 

Required Plugins: - Sonar Scanner - NodeJS - OWASP Dependency Check - Docker Pipeline - Eclipse Temurin installer

 <img width="975" height="449" alt="image" src="https://github.com/user-attachments/assets/6fd1d380-5769-4a66-9c0e-599adbb33248" />

 <img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/c79813b2-c47b-46e6-b20e-453445383dd6" />

Step 4: Configure Jenkins Plugins

SonarQube Setup

•	Launch SonarQube using Docker:

docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/16e80695-198e-40d3-a808-d1bfe5879767" />
 
•	Access SonarQube:
 
 http://<server-ip>:9000

•	Login and update password

•	Generate SonarQube token:

•	Administration → Security → Users → Tokens → Generate Token 

•	Copy token

 <img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/9f053c41-8f87-4f6b-955a-d682d97f3acc" />
 
•	Add token to Jenkins:

•	Jenkins Dashboard → Manage Jenkins → Credentials → Add Secret Text → ID: sonar-token

•	Configure SonarQube server in Jenkins:

•	Manage Jenkins → Configure System → Add SonarQube Server → Name: mysonar

•	Configure SonarQube Scanner:

•	Manage Jenkins → Global Tool Configuration → Add SonarQube Scanner → Name: mysonar

•	Add Quality Gate & Webhook:

•	SonarQube → Administration → Configuration → Webhooks → Create → Name: Jenkins → URL:http://<jenkins-public-ip>:8080/sonarqube-

 <img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/3776df6e-b68a-4f7a-aaf6-d86044c8a863" />

NodeJS, Java, and Dependency Check

•	Install NodeJS and configure JDK 17

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/77d28523-2f46-4e76-9b39-65340e20676d" />

•	Install OWASP Dependency Check plugin in Jenkins

 <img width="979" height="489" alt="image" src="https://github.com/user-attachments/assets/b59f5ccc-544c-4ea7-90ac-6db5ab5f2c59" />

Step 5: Jenkins Declarative Pipeline

pipeline {
    agent any

    tools {
        jdk 'jdk'
        nodejs 'nodejs'
    }

    environment {
        SCANNER_HOME = tool 'sonar'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                 git branch: 'main',
                 url:  'https://github.com/shivanibarya/docker-project.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato
                    """
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
                                odcInstallation: 'dp-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

       stage('Docker Build') {
       steps {
        sh '''
          cd zomato-project1-master
          docker build -t shivanibarya/zomato:v1 .
        '''
    }
}


        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . --severity HIGH,CRITICAL --exit-code 0 > trivyfs.txt'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image shivanibarya/zomato:latest --severity HIGH,CRITICAL'
            }
        }

        stage('Docker Push') {
        steps {
        script {
            withDockerRegistry(
                credentialsId: 'dockerhub',
                url: 'https://index.docker.io/v1/'
            ) {
                sh 'docker push shivanibarya/zomato:latest'
            }
        }
    }
}


        stage('Deploy Container') {
            steps {
                sh '''
                docker rm -f c1 || true
                docker run -itd --name c1 -p 3000:3000 shivanibarya/zomato:v1
                '''
            }
        }

6. Verification

•	Jenkins: Pipeline executed successfully

<img width="975" height="456" alt="image" src="https://github.com/user-attachments/assets/e96f5edc-d716-470f-9492-2ff9ca39aceb" />

<img width="975" height="384" alt="image" src="https://github.com/user-attachments/assets/dbb8a527-46be-435d-9ad6-5a33080f1047" />


•	SonarQube: Code quality verified

•	Trivy: No critical vulnerabilities

•	Docker Hub: Image available at https://hub.docker.com/r/shivanibarya/zomato

<img width="975" height="429" alt="image" src="https://github.com/user-attachments/assets/45f86d92-c142-4360-85e9-57db79f61ec8" />

<img width="975" height="429" alt="image" src="https://github.com/user-attachments/assets/7f66ce23-e433-497d-bfe8-93fa968dcca4" />

•	Application URL: http://<EC2_PUBLIC_IP>:3000

<img width="975" height="505" alt="image" src="https://github.com/user-attachments/assets/4155d298-9315-4428-b1d8-ac49601b98dc" />

 7 .  Key Learnings

•	CI/CD automation with Jenkins

•	Static code analysis using SonarQube

•	Dependency vulnerability scanning using OWASP Dependency Check

•	Container security scanning using Trivy

•	Docker-based application deployment

•	Real-world DevOps pipeline implementation

8 . Conclusion

This project demonstrates a fully automated DevSecOps CI/CD pipeline integrating code quality, dependency security, container security, and Docker-based deployment. It provides hands-on experience in Jenkins, Docker, SonarQube, and Trivy, following industry best practices.
