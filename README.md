# Jenkins Infrastructure Deployment Pipeline
  - These two containers deploy a Jenkins Master and a Jenkins Agents using Docker

# 1. Introduction

#### A communication over SSH

A Jenkins Master can command an Agent to run jobs (cicd pipelines). Both hosts have Java installed on them. The way they communicate is over SSH protocol. The Jenkins Agent is the SSH Server and the Jenkins Master is the SSH Client.

This means the Jenkins Agent and Master must share a pair of SSH public/private keys.

Since the Jenkins Agent is the SHH Server, it holds the SSH public key. On the other hand, since the Jenkins Master is the SSH client, it holds the SSH private key. You can create a pair of SSH public and private keys in any linux machine with the `ssh-keygen` command (see section 3.1 below).

#### Jenkins Agent
The Jenkins Agent is basically a simple SSH server with java installed on it. You can connect any Jenkins Master to it, (Jenkins Masters act as SSH Clients).

To allow the Master to connect to the Agent and schedule jobs in it, you need to:
  - Configure The Jenkins Agent to have the Master's public ssh key.
  - Copy the public key into the `jenkins-agent-ssh/authorized_public_keys` file *before* building and running the Jenkins container.

#### Jenkins Master

 - Build and run the container in `jenkins-master-onprem/`
 - Since the Jenkins Master acts as an SSH client, it needs the ssh private key corresponding to the public key you copied into `jenkins-agent-ssh/authorized_keys`. We explain how to handle that in the Configuration section below.

# 2. Requirements
 - Docker
 - Terraform

# 3. Configuration - STEPS TO DEPLOY

### 3.1. Create the SSH key pair
The SSH key pair is used to establish authentication between the Jenkins Master (SSH client, needs the private key) and the Agent (SSH Server, needs the public key).

You can generate these keys in any linux computer by using the `ssh-keygen` command. The important thing is to **keep the private key secure**.

Let's assume you are creating the SSH key pair in your local linux computer.

Run these commands in your terminal:
```
JENKINS_AGENT_NAME="Agent1"
$ mkdir ~/.ssh; cd ~/.ssh/
$ ssh-keygen -t rsa -m PEM -C "jenkins" -f jenkins${JENKINS_AGENT_NAME}_rsa
```

### 3.2. Configure the public SSH key

Run this command in your terminal:
```
cat ~/.ssh/jenkins${JENKINS_AGENT_NAME}_rsa.pub
# [The output here is a public ssh key]
```
**Copy the output above into the `jenkins-agent-ssh/authorized_keys`**

### 3.3. Build and Run the Jenkins Agent
**TODO: UPDATE THIS WHEN THE TERRAFORM MODULE IS COMPLETED**

Run these commands in your terminal:
```
TAG_NUMBER=0.1
cd jenkins-agent-ssh/
docker build --tag jenkins_agent-ssh_img:$TAG_NUMBER .
docker run --detach --name jenkins-agent-ssh-container-$TAG_NUMBER jenkins_agent-ssh_img:$TAG_NUMBER
cd ..
```

### 3.4. [Optional] Create a Jenkins Master

 If you don't have a Jenkins Master, follow steps here to deploy `jenkins-master-onprem/` as a test to get yourself up and running.

#### 3.4.1. Configure `ENV` variables for the Jenkins Master

You need to configure the `ENV` variables in `jenkins-master-onprem/Dockerfile`, because they are used in the `jcac.yam` file (also called "_Jenkins Configuration As Code_").
 
`jcac.yam` contains all the configurations that otherwise you would need to manually enter in the web UI of the Jenkins Master. `jcac.yam` must not be confused with the `Jenkinsfile`.

`Jenkinsfile` defines the code integration (CI) pipeline and that lives in your Git repository, alongside with your code.

Configure the following variables in `jenkins-master-onprem/Dockerfile` (default values are provided):

```
# SSH Jenkins Agent which we want to connect to
ENV JENKINS_AGENT_NAME="Agent1"
ENV JENKINS_AGENT_IP_ADDR="192.168.9.2"  (the IP of the container you spun up in section 3.3)
ENV JENKINS_AGENT_USER_HOME_DIR="/home/jenkins"
ENV JENKINS_AGENT_REMOTE_DIR="$JENKINS_AGENT_USER_HOME_DIR/jenkins_agent_dir"

# WEB UI LOGIN: Jenkins Master Admin
ENV JENKINS_WEB_UI_ADMIN_USER="admin"
ENV JENKINS_WEB_UI_ADMIN_PASSWD="admin"
ENV JENKINS_WEB_UI_ADMIN_EMAIL="admin@admin.com"

# PIPELINE Variables: Github repository we want Jenkins to connect to
ENV GITHUB_USERNAME="<github-username>"
ENV GITHUB_TOKEN="<github-token>"
ENV GITHUB_REPO_NAME="solutions-terraform-jenkins-gitops"
```

If you are curious about how the variables are used, find them in the `jcac.yaml` file to identify where they are used in the Jenkins web UI.

### 3.4.2. Configure the SSH private key in the jcac.yaml file

The easiest way of configure everything in Jenkins is by using the `jcac.yaml` file. BUT beware that if you are dealing with a production system, you do not want to have the private key in the repository and you should prefer to configure it in the web ui (see section 3.5.1).

Run this command in your terminal:
 ```
 cat ~/.ssh/jenkins${JENKINS_AGENT_NAME}_rsa
 ```

The output from the command above is a private ssh key that looks like this:
```
 # -----BEGIN RSA PRIVATE KEY-----
 # .
 # .
 # . Copy the entire Private key,
 # . including BEGIN and END lines
 # .
 # .
 # -----END RSA PRIVATE KEY-----
```

Copy the private SSH key from the terminal and paste it in the `privateKey` section of the `jcac.yaml` file.

### 3.5. Build and Run the Jenkins Master 
Run these commands in your terminal from `jenkins-master-onprem/` directory:
 ```
TAG_NUMBER="0.1"
docker build --tag jenkins_master_img:$TAG_NUMBER .
docker run --publish 8080:8080 --detach --name jenkins-master-container jenkins_master_img:$TAG_NUMBER 
```

#### 3.5.1. OPTIONAL - Configure the private SSH key in your Jenkins Master's Web UI
Follow these steps from your Jenkins Master's Web UI:

```
1 - Log in to the Web UI of your Jenkins Master
2 - Click on "Credentials"
3 - Find and click on the credential "ssh_private_key_to_connect_to_<Agent-Name>"
4 - Click on "Update"
5 - Click on "Replace"
6 - Paste the private key in the textbox "Enter New Secret Below"
```

### 3.6. Confirm Jenkins Master is connected to the SSH Agent
Follow these steps from your Jenkins Master's Web UI:

```
1 - Log in to the Web UI of your Jenkins Master
2 - Confirm the jenkins-ssh-agent-<Agent-Name> is "Idle" Under "Build Executor Status"
3 - You can also click on the Agent link and then on "Log" to se more details
```

**The `deploy.sh` helper script runs all these steps semi-automatically, but it is better to do it step by step a couple of times before going for the automated way**