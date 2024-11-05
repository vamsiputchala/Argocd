# Jenkins Pipeline for Java based application using Maven, SonarQube, Argo CD,  and Kubernetes

![Screenshot 2023-03-28 at 9 38 09 PM](https://user-images.githubusercontent.com/43399466/228301952-abc02ca2-9942-4a67-8293-f76647b6f9d8.png)

Install Jenkins, configure Docker as agent, set up cicd, deploy applications to k8s and much more.

## AWS EC2 Instance

- Go to AWS Console
- Instances(running)
- Launch instances

<img width="994" alt="Screenshot 2023-02-01 at 12 37 45 PM" src="https://user-images.githubusercontent.com/43399466/215974891-196abfe9-ace0-407b-abd2-adcffe218e3f.png">

### Install Jenkins.

Pre-Requisites:
 - Java (JDK)

### Run the below commands to install Java and Jenkins

Install Java

```
sudo apt update
sudo apt install openjdk-17-jre
```

Verify Java is Installed

```
java -version
```

Now, you can proceed with installing Jenkins

```
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

**Note: ** By default, Jenkins will not be accessible to the external world due to the inbound traffic restriction by AWS. Open port 8080 in the inbound traffic rules as show below.

- EC2 > Instances > Click on <Instance-ID>
- In the bottom tabs -> Click on Security
- Security groups
- Add inbound traffic rules as shown in the image (you can just allow TCP 8080 as well, in my case, I allowed `All traffic`).

<img width="1187" alt="Screenshot 2023-02-01 at 12 42 01 PM" src="https://user-images.githubusercontent.com/43399466/215975712-2fc569cb-9d76-49b4-9345-d8b62187aa22.png">


### Login to Jenkins using the below URL:

http://<ec2-instance-public-ip-address>:8080    [You can get the ec2-instance-public-ip-address from your AWS EC2 console page]

Note: If you are not interested in allowing `All Traffic` to your EC2 instance
      1. Delete the inbound traffic rule for your instance
      2. Edit the inbound traffic rule to only allow custom TCP port `8080`
  
After you login to Jenkins, 
      - Run the command to copy the Jenkins Admin Password - `sudo cat /var/lib/jenkins/secrets/initialAdminPassword`
      - Enter the Administrator password
      
<img width="1291" alt="Screenshot 2023-02-01 at 10 56 25 AM" src="https://user-images.githubusercontent.com/43399466/215959008-3ebca431-1f14-4d81-9f12-6bb232bfbee3.png">

### Click on Install suggested plugins

<img width="1291" alt="Screenshot 2023-02-01 at 10 58 40 AM" src="https://user-images.githubusercontent.com/43399466/215959294-047eadef-7e64-4795-bd3b-b1efb0375988.png">

Wait for the Jenkins to Install suggested plugins

<img width="1291" alt="Screenshot 2023-02-01 at 10 59 31 AM" src="https://user-images.githubusercontent.com/43399466/215959398-344b5721-28ec-47a5-8908-b698e435608d.png">

Create First Admin User or Skip the step [If you want to use this Jenkins instance for future use-cases as well, better to create admin user]

<img width="990" alt="Screenshot 2023-02-01 at 11 02 09 AM" src="https://user-images.githubusercontent.com/43399466/215959757-403246c8-e739-4103-9265-6bdab418013e.png">

Jenkins Installation is Successful. You can now starting using the Jenkins 

<img width="990" alt="Screenshot 2023-02-01 at 11 14 13 AM" src="https://user-images.githubusercontent.com/43399466/215961440-3f13f82b-61a2-4117-88bc-0da265a67fa7.png">

## Install the Docker Pipeline plugin in Jenkins:

   - Log in to Jenkins.
   - Go to Manage Jenkins > Manage Plugins.
   - In the Available tab, search for "Docker Pipeline".
   - Select the plugin and click the Install button.
   - Restart Jenkins after the plugin is installed.
   
<img width="1392" alt="Screenshot 2023-02-01 at 12 17 02 PM" src="https://user-images.githubusercontent.com/43399466/215973898-7c366525-15db-4876-bd71-49522ecb267d.png">

Wait for the Jenkins to be restarted.


## Docker Slave Configuration

Run the below command to Install Docker

```
sudo apt update
sudo apt install docker.io
```
 
### Grant Jenkins user and Ubuntu user permission to docker deamon.

```
sudo su - 
usermod -aG docker jenkins
usermod -aG docker ubuntu
systemctl restart docker
```

Once you are done with the above steps, it is better to restart Jenkins.

```
http://<ec2-instance-public-ip>:8080/restart
```

The docker agent configuration is now successful.


## Next Steps

### Configure a Sonar Server locally

```
apt install unzip
adduser sonarqube
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
unzip *
chmod -R 755 /home/sonarqube/sonarqube-9.4.0.54424
chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-9.4.0.54424
cd sonarqube-9.4.0.54424/bin/linux-x86-64/
./sonar.sh start
```

 Now you can access the `SonarQube Server` on `http://<ip-address>:9000` 

## Jenkins Pipeline Documentation
Prerequisites
## 1.Docker: 
   Docker must be installed on the Jenkins agent and have permission to access the Docker daemon (/var/run/docker.sock).
## 2.Jenkins Plugins:
- Docker Pipeline: Required to use Docker commands within the pipeline.
- Git Plugin: For cloning the repository.
- SonarQube Scanner: For static code analysis with SonarQube.
## Credentials:
- GitHub Token: Named github in Jenkins, with permissions to clone and push to the repository.
- SonarQube Token: Named sonarqube, with permissions to submit analysis to the SonarQube server.
- Docker Hub Credentials: Named docker-cred in Jenkins, with permissions to push images to Docker Hub.

## Pipeline Breakdown
The Jenkins pipeline is a declarative pipeline using a Docker agent. Below are explanations for each stage.

## 1. Agent Configuration
The pipeline runs on a Docker agent, specifically vamsi1011/maven-vamsi-docker-agent:v1, which has Maven and Docker installed 
```
agent {
    docker {
      image 'vamsi1011/maven-vamsi-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock' // Mount Docker socket for Docker access
    }
  }
```

- image: Docker image to be used as the Jenkins agent.
- args: The --user root allows root access, and -v /var/run/docker.sock:/var/run/docker.sock provides access to the host's Docker daemon.
## 2. Stages
Each stage represents a different phase in the CI/CD pipeline.
## Stage 1: Checkout
This stage checks out the code from the GitHub repository.
```
stage('Checkout') {
  steps {
    git branch: 'main', url: 'git@github.com:vamsiputchala/Argocd.git', credentialsId: 'github'
  }
}
```
- branch: The branch to clone (in this case, main).
- url: URL of the GitHub repository.
- credentialsId: github credentials are used for authentication
## Stage 2: Build and Test
This stage compiles the project, runs tests, and generates a JAR file using Maven.
```
stage('Build and Test') {
  steps {
    sh 'ls -ltr'
    sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn clean package'
  }
}
```
- ls -ltr: Lists files to confirm the working directory.
- mvn clean package: Builds the project and creates a JAR file in the spring-boot-app directory.

  
  ![Screenshot 2024-11-05 190436](https://github.com/user-attachments/assets/2dbac25d-7d5a-47c5-bb15-0bca95dac70b)

## Stage 3: Static Code Analysis
In this stage, SonarQube performs static code analysis to detect code smells, vulnerabilities, and other quality issues.
```
stage('Static Code Analysis') {
  environment {
    SONAR_URL = "http://3.144.199.42:9000"
  }
  steps {
    withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')]) {
      sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}'
    }
  }
}
```
- SONAR_URL: URL of the SonarQube server.
- withCredentials: Injects the SonarQube token (SONAR_AUTH_TOKEN) for authentication.
- mvn sonar
  : Executes the SonarQube analysis with the provided URL and token.

  
![Screenshot 2024-11-05 190531](https://github.com/user-attachments/assets/e39393c9-b295-4384-96d6-b83ef037d307)


## Stage 4: Build and Push Docker Image
This stage builds a Docker image from the project and pushes it to Docker Hub
```
stage('Build and Push Docker Image') {
  environment {
    DOCKER_IMAGE = "vamsi1011/ultimate-cicd:${BUILD_NUMBER}"
  }
  steps {
    script {
      sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && docker build -t ${DOCKER_IMAGE} .'
      def dockerImage = docker.image("${DOCKER_IMAGE}")
      docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
        dockerImage.push()
      }
    }
  }
}
```
- DOCKER_IMAGE: The Docker image tag, using Jenkins’ BUILD_NUMBER for versioning.
- docker build: Builds the Docker image from the spring-boot-app directory.
- docker.withRegistry: Logs in to Docker Hub with docker-cred and pushes the image.


  ![Screenshot 2024-11-05 190655](https://github.com/user-attachments/assets/99876a65-0cdc-4c74-84af-d3e8e204dd6e)

## Stage 5: Update Deployment File
This stage updates the Kubernetes deployment file with the new Docker image tag, commits the change to Git, and pushes the updated file to GitHub.
```
stage('Update Deployment File') {
  environment {
    GIT_REPO_NAME = "Argocd"
    GIT_USER_NAME = "vamsiputchala"
  }
  steps {
    withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
      sh '''
        git config user.email "vamsiputchala@gmail.com"
        git config user.name "vamsiputchala"
        BUILD_NUMBER=${BUILD_NUMBER}
        sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests/deployment.yml
        git add java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests/deployment.yml
        git commit -m "Update deployment image to version ${BUILD_NUMBER}"
        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
      '''
    }
  }
}
```
- GITHUB_TOKEN: Token for authenticating the Git push operation.
- sed -i: Replaces replaceImageTag in the deployment YAML file with the BUILD_NUMBER.
- git push: Pushes the updated YAML file to the main branch.

  
  ![Screenshot 2024-11-05 190739](https://github.com/user-attachments/assets/5af9c18b-a0df-46a4-9a43-b9e2193e0cc8)

  ## Finally deployed in Argocd  pods:
  
  
  ![Screenshot 2024-11-05 191413](https://github.com/user-attachments/assets/a7fadd0f-85dc-4cf6-ab75-0e0a119246f6)

## Summary
## This Jenkins pipeline automates the following steps:

## 1.Clone the Code from GitHub.
## 2.Build and Test the Maven project.
## 3.Perform Static Code Analysis using SonarQube.
## 4.Build and Push a Docker image to Docker Hub.
## 5.Update the Deployment File in the GitHub repository.
This setup enables Continuous Integration and Continuous Deployment (CI/CD) for a Spring Boot application with a Kubernetes deployment setup managed by Argo CD. Each build updates the image tag in the deployment manifest, which Argo CD can then automatically synchronize with the Kubernetes cluster.



