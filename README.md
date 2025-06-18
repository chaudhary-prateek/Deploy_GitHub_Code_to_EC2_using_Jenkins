# 🚀 CI/CD Pipeline: Deploy GitHub Code to EC2 using Jenkins

This guide helps you set up a complete CI/CD pipeline where code from your GitHub repository is automatically deployed to an EC2 server using Jenkins.

--------------------------------------------------------------

## 📌 1. GitHub Setup

### 🔑 1.1 Generate Personal Access Token (PAT)

1. Go to **GitHub → Settings → Developer Settings → Personal access tokens → Tokens (classic)**.
2. Click **Generate new token (classic)**.
3. Fill in:
   - **Note**: `Jenkins-CICD-Token`
   - **Expiration**: `1 year` (or as needed)
   - **Scopes**: ✅ `repo`
4. Click **Generate token** and **copy it immediately**.

--------------------------------------------------------------

### 🔔 1.2 Add Webhook to GitHub Repo
1. Go to your repo:  
   `https://github.com/<your-org-or-username>/<your-repo>`
2. Navigate to **Settings → Webhooks → Add Webhook**.
3. Fill in:
   - **Payload URL**: `http://<JENKINS_PUBLIC_IP>:8080/github-webhook/`
   - **Content type**: `application/json`
   - **Event type**: Just the push event
4. Click **Add webhook**.

-------------------------------------------------------------

### ✨ 1.3 Add EC2 SSH Public Key to GitHub
**On EC2 Instance:**
``bash

ssh-keygen -t rsa -b 4096 -C "deploy@ec2"
cat ~/.ssh/id_rsa.pub

-------
**On GitHub:
Go to: Repo → Settings → Deploy keys → Add deploy key
Add:**

Title: EC2-Deploy-Key

Key: Paste the copied public key

✅ Allow write access (if needed)

--------------------------------------------------------------

**🧭 1.4 Configure GitHub SSH Access on EC2**

cd /var/www/html/<your-project-folder>
git remote set-url origin git@github.com:<your-org-or-username>/<your-repo>.git
ssh-keyscan github.com >> ~/.ssh/known_hosts
ssh -T git@github.com

--------------------------------------------------------------

**⚙️ 2. Jenkins Setup**

📦 2.1 Install Required Plugins
Git plugin

GitHub plugin

Pipeline plugin

SSH Agent plugin

--------------------------------------------------------------

**🔐 2.2 Add GitHub PAT to Jenkins
Path: Manage Jenkins → Credentials → System → Global credentials**

Kind: Secret text

ID: github_access_token

Secret: Paste your GitHub PAT

--------------------------------------------------------------

**🔑 2.3 Add EC2 SSH Key to Jenkins**
Kind: SSH Username with private key

Username: ubuntu (or your EC2 user)

Private key: Paste your .pem or EC2 private key

ID: ec2_ssh_key

---------------------------------------------------------------

**🧲 2.4 Create Jenkins Pipeline Job**
Go to Jenkins Dashboard → New Item

Name: Web_CICD (or your project name)

Type: Pipeline → OK

Configuration:

Definition: Pipeline script from SCM

SCM: Git

Repository URL: https://github.com/<your-org-or-username>/<your-repo>.git

Credentials: github_access_token

Branch: main

Script Path: Jenkinsfile

---------------------------------------------------------------

**🖥️ 3. EC2 Server Setup 🔠 3.1 Install Git**

sudo apt update
sudo apt install git -y

---------------------------------------------------------------

**🔒 3.2 Enable Jenkins SSH Access to EC2
In EC2 Security Group:  ✅ Allow port 22 from Jenkins server IP**

On EC2:

echo "ssh-rsa AAAAB3Nza..." >> ~/.ssh/authorized_keys
ssh ubuntu@<EC2_PUBLIC_IP>
ssh-keyscan github.com >> ~/.ssh/known_hosts

---------------------------------------------------------------

**🛠️ 4. Jenkinsfile Example**

```groovy
pipeline {
    agent any

    environment {
        REMOTE_USER = "ubuntu"
        REMOTE_HOST = "<EC2_PUBLIC_IP>"
        PROJECT_DIR = "/var/www/html/<your-project-folder>"
        SSH_CREDENTIALS_ID = 'ec2_ssh_key'
    }

    stages {
        stage('Pull Latest Code') {
            steps {
                sshagent (credentials: [env.SSH_CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} '
                            cd ${PROJECT_DIR} &&
                            git stash &&
                            git pull origin main
                        '
                    """
                }
            }
        }
    }
}
