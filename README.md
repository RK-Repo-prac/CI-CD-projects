# Jenkins CI/CD Pipeline for deploying a Java Application using maven build tool , ArgoCD , Kubernetes ,Docker and SonarQube

## Infrastructure Architecture 

* I chose t2.large instance type for the master node to support installation and running of multiple tools like SonarQube, Docker, and Jenkins that will require significant compute and memory resources.
* I provisioned Docker agent nodes to enable on-demand scalability. Using Docker containers allows the build infrastructure to scale up when runs are executing and scale down when idle, maximizing resource utilization efficiency.
* I implemented Argo CD for application deployments in order to automate the process of deploying code changes into production. Argo CD syncs the live state against the Git repository state and automatically deploys any changes committed to the repo to Kubernetes clusters. This eliminates the need for manual deployments for every code change.

## Here are the step-by-step details to set up an end-to-end Jenkins pipeline for a Java application using SonarQube, Argo CD, and Kubernetes:

### Step 1:

Launch a EC2 "Ubuntu Instance" of type "t2.large" in AWS

### Step 2:

We plan to use Jenkins, which is written in Java, on an Ubuntu machine. To enable Jenkins to work, we need to install Java runtime. We use the apt package manager to install both Java and Jenkins on the Ubuntu machine.

```
sudo apt update
sudo apt install openjdk-11-jre
java -version

```

Next we Install Jenkins:
```
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
/usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update

sudo apt-get install jenkins

```

### Step 3:

Install DOcker on Master Node and in Jenkins add Docker Pipeline Plugin.

```
sudo apt update
sudo apt install docker.io -y

```

### Step 4:

Install Sonarqube plugin in the Jenkins and once it is done in the Master node install SonarQube.

```
apt install unzip

adduser sonarqube

wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip

unzip *

mv sonarqube-9.4.0.54424 sonarqube

chmod -R 755 /home/sonarqube/sonarqube

chown -R sonarqube:sonarqube /home/sonarqube/sonarqube

cd sonarqube/bin/linux-x86-64/
./sonar.sh start


```
Initially, the default admin username and password can be used to log into SonarQube. An authentication token should then be generated within SonarQube and added to Jenkins credentials. This token will be used by Jenkins pipelines to authenticate with SonarQube when triggering analysis, instead of using the default admin credentials.


### Step 5

Define and Configure Jenkins Pipeline Stages

Stage Checkout - Check out code from Git repository 
Stage Build and Test - Build project and run tests
Stage Static Code Analysis - Run static code analysis with SonarQube
Stage Build and Push Docker Image - Build Docker Stage image and push to registry
Stage Update Deployment File - Update Kubernetes deployment file with new image tag and commit changes

Note: Here I skipped the checkout part since ,Jenkins project configuration has SCM details to clone a git repo.
When Jenkins starts the pipeline, it first clones the repo to its own host workspace,then Jenkins launches the Docker agent, and mounts this workspace into the container. So the git repo already exists in the mounted location inside docker
Therefore, pipeline stages do not need to explicitly checkout again

### Step 6

Install Minikube and Kubectl on a seperate Instance,

Go to https://operatorhub.io/ and install ArgoCd Operator on the same instance where our Kubernetes cluster is present.

Steps to install ArgoCD

```
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.26.0/install.sh | bash -s v0.26.0

kubectl create -f https://operatorhub.io/install/argocd-operator.yaml


kubectl get csv -n operators

```

After the Operator is running deploy the below yaml manifest 

```
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: example-argocd
  labels:
    example: basic
spec: {}

```

After Doing the above steps our ArgoCD controller is installed onto the machine. Use the below commands to access it on the browser

```
kubectl get svc
#Edit the below manifest to change it from Type ClusterIP to LoadBalancer mode

kubectl edit svc example-argocd-server

# Below commands are to Fetch the password

kubectl get secret

kubectl edit example example-argocd-cluster

```

ArgoCD operates by integrating with Jenkins during the build and deploy stages. In Jenkins, as part of the build process, we dynamically update the deployment.yml file with the latest build tag and commit these changes to the repository. Subsequently, this modification triggers the ArgoCD controller, prompting it to deploy the updated deployment.yml file to the designated Kubernetes cluster. It's worth noting that, in this setup, we are utilizing the same Kubernetes cluster for both Jenkins and ArgoCD.



