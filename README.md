# End-to-End CI/CD Pipeline for a Java Application using Jenkins, Maven, SonarQube, Docker, Argo CD, and Kubernetes

## 1. Overview of the Pipeline

The CI/CD pipeline integrates tools such as:
- Jenkins: For Continuous Integration (CI) tasks.
- Maven: To build and manage the Java project.
- SonarQube: For static code analysis and ensuring code quality.
- Docker: To containerize the application.
- Argo CD: To handle Continuous Delivery (CD) with Kubernetes.
- Kubernetes: To orchestrate and manage deployments.

**Workflow**
1. Code Commit: Developers commit code changes to a Git repository hosted on GitHub.
2. Jenkins Build: Jenkins is triggered to build the code using Maven. Maven builds the code and runs unit tests.
3. Code Analysis: Sonar is used to perform static code analysis to identify any code quality issues, security vulnerabilities, and bugs.
4. Security Scan: AppScan is used to perform a security scan on the application to identify any security vulnerabilities.
5. Deploy to Dev Environment: If the build and scans pass, Jenkins create a Docker image and push it to a container registry.
6. Continuous Delivery: ArgoCD is used to manage continuous delivery. ArgoCD watches the Git repository and automatically deploys new changes (Argo CD pulls the latest image and updates Kubernetes manifests) to the development environment as soon as they are committed.
7. Promote to Production: When the code is ready for production, it is manually promoted using ArgoCD to the production environment.
8. Monitoring: The application is monitored for performance and availability using Kubernetes tools and other monitoring tools.

## 2. Tools and Configuration

### 2.1 Jenkins
- Jenkins acts as the central orchestrator for the CI process.
- Trigger Mechanism: Webhooks are set up to trigger Jenkins pipelines automatically upon code commits.
- **Pipeline Stages:**
    - Build Stage: Maven compiles the code and runs unit tests.
    - Static Code Analysis: SonarQube performs quality checks.
    - Docker Image Creation: Docker builds an image and pushes it to a registry.
    - Manifest Update: Kubernetes manifests are updated (via shell scripts or Argo Image Updater).

### 2.2 Maven
- Maven is used for building the Java application.
- Key Task: Executes `mvn clean package` to create the build artifact (e.g., `.jar` file).
- `mvn clean package` vs `mvn clean install:`
    - `clean package`: Compiles and packages the code.
    - `clean install`: Additionally pushes the artifact to a repository like Nexus.

### 2.3 SonarQube
- SonarQube scans the code for bugs, vulnerabilities, and code smells.
- Authentication: A token is generated in SonarQube and stored in Jenkins for secure communication.
- Workflow:
    - Jenkins sends the code to SonarQube for analysis.
    - SonarQube runs checks and sends a report back.
    - Jenkins decides whether to proceed based on the SonarQube report.

### 2.4 Docker
- Docker creates a container image of the Java application using the build artifact.
- The image is tagged (e.g., `v1.0.0`) and pushed to a container registry (e.g., Docker Hub, AWS ECR).
- Command: `docker build -t <image_name>:<tag>`
- Push Image: `docker push <registry>/<image_name>:<tag>`

### 2.5 Argo CD
- Argo CD handles Continuous Delivery.
- It watches the Kubernetes manifest repository for changes.
- Process:
    - Argo Image Updater checks for new image version.
    - The manifest repository (e.g., `deployment.yaml`) is updated with the new image tag.
    - Commits updated manifests back to the Git repository.
    - Argo CD monitors the manifests repository and detects changes.
    - Argo CD applies the updated manifests to the Kubernetes cluster.
   
### 2.6 Kubernetes
- Kubernetes orchestrates the application deployment.
- Deployments are described in manifests (`deployment.yaml`, `service.yaml`, etc.).

## 3. CI/CD Workflow

### 3.1 Continuous Integration (CI)
- Trigger Pipeline: Jenkins is triggered via Git webhook.
- Build: Maven compiles and package the application into a JAR file.
- Test: JUnit and Mockito for running unit tests.
- Code Quality Analysis: SonarQube scans the code for issues.
- Docker Build: A Docker image is built and pushed to a container registry.
- Notification: If any stage fails, email or Slack notifications are sent.

### 3.2 Continuous Delivery (CD)
- Update Manifests: The Docker image tag in Kubernetes manifests is updated via Argo Image Updater.
- Apply Manifests: Argo CD picks up the changes and applies them to the cluster.
- Kubernetes Deployment: The updated application is deployed as pods on Kubernetes.

## 4. Key Interview Questions from the Pipeline

1. **How does Jenkins get notified of code changes?**
    - Via webhooks configured in GitHub.
2. **Why use Docker agents in Jenkins pipelines?**
    - To reduce dependency installation overhead and improve scalability.
    - Each stage runs in isolated Docker containers, avoiding dependency conflicts.
    - Docker agents are lightweight and easy to scale.
    - In Jenkins, when using Docker agents (ephemeral containers as Jenkins agents), they are often designed to spin up, execute the task, and then terminate. This behavior is intentional and aligns with the principles of ephemeral infrastructure and resource efficiency.
3. **What is the difference between Continuous Integration and Continuous Delivery?**
    - CI: Ensures code is built, tested, and packaged.
    - CD: Ensures the deployment process is automated.
4. **Why use Argo CD instead of Ansible for CD?**
    - Argo CD follows the GitOps approach, ensuring better scalability and version control.
    - Ensures auditability and traceability of all changes.
    - Provides a single source of truth in the Git repository.
    - Prevents manual changes to Kubernetes clusters.
5. **What is the role of SonarQube in the pipeline?**
    - To analyze code quality and ensure compliance with standards.
6. **What is the purpose of Kubernetes manifests?**
    - Declaratively define application resources.

## 5. Benefits of this CI/CD Setup

- Automation: End-to-end automation from code commit to deployment.
- Scalability: Kubernetes ensures applications can scale dynamically.
- Reliability: Argo CD guarantees that deployments remain consistent.
- Improved Code Quality: SonarQube helps detect vulnerabilities early.
- Efficient Collaboration: Jenkins pipelines and version-controlled manifests simplify teamwork.

## Implementaton Steps

![Screenshot 2023-03-28 at 9 38 09 PM](https://user-images.githubusercontent.com/43399466/228301952-abc02ca2-9942-4a67-8293-f76647b6f9d8.png)

### Step 1: Setup the Spring Boot Application on a Aws Ec2-instance
- Choose Ubuntu machine.
- Open inbound traffic rules for necessary ports (e.g., `22`, `80`, `8010`, `8080`).
- SSH into the EC2 instance: 
    `ssh -i <your-key.pem> ubuntu@<ec2-ip>`
```bash
# Update system packages:
sudo apt update && sudo apt upgrade -y

# Install required packages: 
sudo apt install git docker.io maven openjdk-17-jre -y

sudo systemctl enable docker
sudo systemctl start docker
```
```bash
# This is a simple Sprint Boot based Java application that can be built using Maven. Sprint Boot dependencies are handled using the `pom.xml` at the root directory of the repository.

# This is a MVC architecture based application where controller returns a page with title and message attributes to the view.

# Checkout the repo and move to the directory
git clone https://github.com/aqeeladil/springboot-complete-cicd
cd sprint-boot-app

# Execute the Maven targets to generate the artifacts
mvn clean package

# The above maven target stores the artifacts to the `target` directory. You can either execute the artifact (or) run it as a Docker container.
# To avoid issues with Java versions and other dependencies, I would recommend the docker way.

# The Artifact way
`java -jar target/spring-boot-web.jar`
# Access the application on http://localhost:8080

# OR
# The Docker way
docker build -t ultimate-cicd-pipeline:v1 .
docker run -d -p 8010:8080 -t ultimate-cicd-pipeline:v1
# Access the application on `http://<ec-ip-address>:8010`
```

### Step 2: Launch a 2nd Ec2 Instance for CICD setup
- Choose Ubuntu with t2.large (2 CPUs, 8GB RAM).
- Open inbound traffic rules for necessary ports (e.g., `22`, `80`, `8080`, `9000`).
- SSH into the EC2 instance: 
    `ssh -i <your-key.pem> ubuntu@<ec2-ip>`
```bash
# Update system packages:
sudo apt update && sudo apt upgrade -y

# Install Required Dependencies:
sudo apt install apt-transport-https ca-certificates git curl wget unzip   software-properties-common -y
```

### Step 3: Jenkins Setup (Continuous Integration Tool)
```bash
# Install Java
sudo apt install openjdk-17-jre -y

# Add Jenkins Repository and Install Jenkins
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt update -y
sudo apt install jenkins -y
ps -ef | grep jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# By default, Jenkins will not be accessible to the external world due to the inbound traffic restriction by AWS. Open port 8080 in the inbound traffic rules as show below.
# Access Jenkins via http://<EC2_Public_IP>:8080.

# Retrieve the initial admin password:
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Create first Admin User or Skip the step [If you want to use this Jenkins instance for future use-cases as well, better to create admin user]

# Install required plugins.
# Git Plugin, Docker Pipeline Plugin, Maven Integration Plugin, SonarQube Scanner Plugin, Kubernetes Continuous Deploy plugin
# Go to Manage Jenkins → Manage Plugins → Available Plugins and install them.
# Restart Jenkins after the plugins are installed.
# http://<ec2-instance-public-ip>:8080/restart

# Create a new pipeline job (New Item -> Pipeline) and configure it with the Git repository URL for the Java application.

# Add Github credentials (Settings -> Developer Settings -> Personal Access Token -> Generate Token)
# Go to Manage Jenkins → Manage Credentials → Add Credentials

# JENKINS PIPELINE STAGES:
# Stage 1: Use the Git plugin to check out the source code from the Git repository.
# Stage 2: Use the Maven Integration plugin to build the Java application.
# Stage 3: Use the JUnit and Mockito plugins to run unit tests.
# Stage 4: Use the SonarQube plugin to analyze the code quality of the Java application.
# Stage 5: Use the Maven Integration plugin to package the application into a JAR file.
# Stage 6: Use the Kubernetes Continuous Deploy plugin to deploy the application to a test environment using Helm.
# Stage 7: Use a testing framework like Selenium to run user acceptance tests on the deployed application.
# Stage 8: Use Argo CD to promote the application to a production environment.
```

### Step 4: Maven & Docker Agent Configuration
- Maven is already installed as part of this container image `abhishekf5/maven-abhishek-docker-agent:v1` already configured in the JenkinsFile.

### Step 5: SonarQube Setup (Static Code Analysis)
```bash
# Add SonarQube user
sudo adduser sonarqube
sudo su - sonarqube

# Download SonarQube
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.0.0.45539.zip
unzip sonarqube-9.0.0.45539.zip
chmod -R 755 /home/sonarqube/sonarqube-9.0.0.45539
chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-9.0.0.45539
cd sonarqube-9.0.0.45539/bin/linux-x86-64/

# Start SonarQube
./sonar.sh start

# Access SonarQube: http://<EC2_Public_IP>:9000
# Default Credentials: Username/Password = admin/admin

# Generate SonarQube Token for Jenkins Integration.
# Go to My Account → Security → Generate Token
# Copy the token and save it.

# Configure SonarQube Credentials in Jenkins
# Go to Manage Jenkins → Manage Credentials → Add Credentials
# Add SonarQube Token under Global Credentials.
```

### Step 6: Install Docker
```bash
sudo apt install docker.io
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
sudo systemctl restart docker
sudo systemctl restart jenkins
# Another way to restart jenkins: `http://<ec2-instance-public-ip>:8080/restart`

# Configure DockerHub Credentials in Jenkins
# Go to Manage Jenkins → Manage Credentials → Add Credentials
# Add DockerHub Token under Global Credentials.
```

### Step 7: Minikube and Argo CD Setup (Continuous Delivery) (Setup it using a 3rd ec2-instance)
```bash
# Launch an ec2 instance and allow inbound traffic on ports 22, 80, 443, 8080
# SSH into the instance
ssh -i <your-key.pem> ubuntu@<ec2-ip>

# Update the System
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl wget

# Install Docker: Minikube uses Docker as its default driver.
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker

# Install Minikube
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube /usr/local/bin/
rm minikube

# Install Kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/
rm kubectl
kubectl get nodes

# Start Minikube Cluster
minikube start --driver=docker
minikube status

# Install Argo CD using Operators (Reference: https://operatorhub.io/operator/argocd-operator)
# Install Operator Lifecycle Manager (OLM), a tool to help manage the Operators running on your cluster.
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.30.0/install.sh | bash -s v0.30.0
# Install the operator by running the following command: This Operator will be installed in the "operators" namespace and will be usable from all namespaces in the cluster.
kubectl create -f https://operatorhub.io/install/argocd-operator.yaml
kubectl get csv -n operators
kubectl get pods -n operators
vim argocd-basics.yaml
```
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: example-argocd
  labels:
    example: basic
spec: {}
```
```bash
kubectl apply -f argocd-basics.yaml
kubectl get pods
kubectl get svc

# Update the Argo CD server service to from type: "ClusterIP" to "NodePort".
kubectl edit svc example-argocd-server
# Verify 
kubectl get svc

# Access ArgoCD UI
kubectl get pods
minikube service argocd-server
minikube service list
# Open the provided URL in your browser

# Username: admin
# Retrieve the default password
kubectl get secret
kubectl edit secret example-argocd-cluster 

# Copy the encrypted password from the above file and paste it here
echo <encrypted-password> | base64 -d

# Login to the ArgoCD UI and craete a new application providing the github repo url and directory path for manifest files.

# Sync the application and verify
kubectl get deploy
kubectl get pods
```

### Step 8: Validate the Workflow
- Commit changes to the Git repository.
- Jenkins pipeline triggers automatically.
- SonarQube performs code quality checks.
- Docker builds and pushes the image.
- Argo CD updates Kubernetes manifests and deploys the app.
- Validate the deployment in minikube cluster.
- Monitor deployment logs.
- Verify the running application.
- Fix any issues that arise.


