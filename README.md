
**DevSecOps: Building Security Into Every Step of the Pipeline (Part I)**

In this project we would be deploying Netflix Clone application using Terraform, Jenkins (CI/CD) pipeline, Dockerto implement an end-to-end DevSecOps  deployed on AWS,  Security tools like like SonarQube, OWASP Dependency Check and Trivy would be used. 


**Step 1: Launch an EC2 Instance and install Jenkins, SonarQube, Docker and Trivy**
We would be making use of Terraform to launch the EC2 instance. We would be adding a script as userdata for the installation of Jenkins, SonarQube, Trivy and Docker.



**Step 2: Access Jenkins at port 8080 and install required plugins**
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
Blue Ocean



**Step 3: Set up SonarQube**
For the SonarQube Configuration, first access the Sonarqube Dashboard using the url http://elastic_ip:9000

Create the token Administration -> Security -> Users -> Create a token

Add this token as a credential in Jenkins

Go to Manage Jenkins -> System -> SonarQube installation Add URL of SonarQube and for the credential select the one added in step 2.

Go to Manage Jenkins -> Tools -> SonarQube Scanner Installations -> Install automatically.



**Step 4: Set up OWASP Dependency Check**
Go to Manage Jenkins -> Tools -> Dependency-Check Installations -> Install automatically



**Step 5: Set up Docker for Jenkins**
Go to Manage Jenkins -> Tools -> Docker Installations -> Install automatically

And then go to Manage Jenkins -> Credentials -> System -> Global Credentials -> Add credentials. Add username and password for the docker registry (You need to create an account on Dockerhub).



**Step 6: Create a pipeline in order to build and push the dockerized image securely using multiple security tools**
Go to Dashboard -> New Item -> Pipeline

Use the code below for the Jenkins pipeline.

```python
pipeline {
    agent any
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'hedobriggs/netflix-clone-feb3'
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        TMDB_API_KEY = credentials('tmdb-api-key')
    }
    
    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/hedobriggs/netflix.git'
            }
        }
        
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=netflix-clone-feb3 \
                        -Dsonar.projectKey=netflix-clone-feb3'''
                }
                echo "SonarQube analysis submitted"
            }
        }
        
        stage("Quality Gate") {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        try {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                echo "Quality gate failed with status: ${qg.status}"
                                echo "Continuing pipeline execution..."
                            } else {
                                echo "Quality gate passed!"
                            }
                        } catch (Exception e) {
                            echo "Quality gate check failed: ${e.message}"
                            echo "Continuing pipeline execution..."
                        }
                    }
                }
            }
        }
        
        stage('OWASP Dependency Check') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_KEY')]) {
                        dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_KEY} --format HTML --format XML",
                                       odcInstallation: 'OWASP DP-Check'
                        
                        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                    }
                }
            }
        }
        
        stage('Trivy Filesystem Scan') {
            steps {
                script {
                    sh "trivy fs . > trivyfs.txt"
                    echo readFile('trivyfs.txt')
                }
            }
        }
        
        stage("Build Docker Image") {
            steps {
                script {
                    sh """
                        docker build \
                        --build-arg API_KEY=${TMDB_API_KEY} \
                        -t ${DOCKER_IMAGE}:${DOCKER_TAG} \
                        -t ${DOCKER_IMAGE}:latest \
                        .
                    """
                }
            }
        }
        
        stage("Trivy Image Scan") {
            steps {
                script {
                    sh "trivy image ${DOCKER_IMAGE}:${DOCKER_TAG} > trivyimage.txt"
                    echo readFile('trivyimage.txt')
                }
            }
        }
        
        stage("Push to Docker Hub") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh """
                            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage("Deploy to EC2") {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no -i \$SSH_KEY ec2-user@18.135.159.182 '
                                
                                echo "🔍 Checking for containers using port 80..."
                                OLD_CONTAINER=\$(docker ps -q --filter "publish=80")
                                if [ ! -z "\$OLD_CONTAINER" ]; then
                                    echo "🛑 Stopping container using port 80: \$OLD_CONTAINER"
                                    docker stop \$OLD_CONTAINER || true
                                    docker rm \$OLD_CONTAINER || true
                                else
                                    echo "✔ No container currently using port 80"
                                fi
                                
                                echo "📦 Pulling image from Docker Hub: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                                docker pull ${DOCKER_IMAGE}:${DOCKER_TAG}
                                
                                echo "🚀 Starting new container with image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                                docker run -d --name netflix-app-${DOCKER_TAG} -p 80:3000 ${DOCKER_IMAGE}:${DOCKER_TAG}
                                
                                echo "✅ Deployment complete! Running: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                            '
                        """
                    }
                }
            }
        }
        
        stage("Cleanup") {
            steps {
                script {
                    sh """
                        docker rmi ${DOCKER_IMAGE}:${DOCKER_TAG} || true
                        docker rmi ${DOCKER_IMAGE}:latest || true
                    """
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: '**/trivyfs.txt, **/trivyimage.txt, **/dependency-check-report.*',
                            allowEmptyArchive: true
            cleanWs()
        }
    }
}```
