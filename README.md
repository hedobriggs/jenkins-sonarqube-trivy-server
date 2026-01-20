Deploying a Netflix Clone on Kubernetes using DevSecOps methodology
In this project we would be deploying Netflix Clone application on an EKS cluster using DevSecOps methodology. We would be making use of security tools like SonarQube, OWASP Dependency Check and Trivy. We would also be monitoring our EKS cluster using monitoring tools like Prometheus and Grafana. Most importantly we will be using ArgoCD for the Deployment.

Step 1: Launch an EC2 Instance and install Jenkins, SonarQube, Docker and Trivy
We would be making use of Terraform to launch the EC2 instance. We would be adding a script as userdata for the installation of Jenkins, SonarQube, Trivy and Docker.

Step 2: Access Jenkins at port 8080 and install required plugins
Install the following plugins:

NodeJS
Eclipse Temurin Installer
SonarQube Scanner
OWASP Dependency Check
Docker
Docker Commons
Docker Pipeline
Docker API
docker-build-step
Step 3: Set up SonarQube
For the SonarQube Configuration, first access the Sonarqube Dashboard using the url http://elastic_ip:9000

Create the token Administration -> Security -> Users -> Create a token

Add this token as a credential in Jenkins

Go to Manage Jenkins -> System -> SonarQube installation Add URL of SonarQube and for the credential select the one added in step 2.

Go to Manage Jenkins -> Tools -> SonarQube Scanner Installations -> Install automatically.

Step 4: Set up OWASP Dependency Check
Go to Manage Jenkins -> Tools -> Dependency-Check Installations -> Install automatically
Step 5: Set up Docker for Jenkins
Go to Manage Jenkins -> Tools -> Docker Installations -> Install automatically

And then go to Manage Jenkins -> Credentials -> System -> Global Credentials -> Add credentials. Add username and password for the docker registry (You need to create an account on Dockerhub).

Step 6: Create a pipeline in order to build and push the dockerized image securely using multiple security tools
Go to Dashboard -> New Item -> Pipeline

Use the code below for the Jenkins pipeline.

pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/gauri17-pro/nextflix.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'OWASP DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                script {
                    try {
                        sh "trivy fs . > trivyfs.txt" 
                    }catch(Exception e){
                        input(message: "Are you sure to proceed?", ok: "Proceed")
                    }
                }
            }
        }
        stage("Docker Build Image"){
            steps{
                   
                sh "docker build --build-arg API_KEY=2af0904de8242d48e8527eeedc3e19d9 -t netflix ."
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image netflix > trivyimage.txt"
                script{
                    input(message: "Are you sure to proceed?", ok: "Proceed")
                }
            }
        }
        stage("Docker Push"){
            steps{
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker'){   
                    sh "docker tag netflix gauris17/netflix:latest "
                    sh "docker push gauris17/netflix:latest"
                    }
                }
            }
        }
    }
}
