# 🚀 Jenkins CI/CD — EC2 + GitHub + S3 Complete Guide

> **Setup:** Everything runs in AWS Cloud. No local machine needed after initial EC2 launch.  
> **Flow:** Push code to GitHub → Jenkins auto-detects → Builds → Uploads artifact to S3  
> **Project Used:** A simple Node.js web app hosted on GitHub as reference

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [What You Will Need](#what-you-will-need)
3. [Step 1 — Create IAM Role for EC2](#step-1--create-iam-role-for-ec2)
4. [Step 2 — Create S3 Bucket](#step-2--create-s3-bucket)
5. [Step 3 — Launch EC2 Instance](#step-3--launch-ec2-instance)
6. [Step 4 — Security Group Rules](#step-4--security-group-rules)
7. [Step 5 — Connect to EC2 via Browser (EC2 Instance Connect)](#step-5--connect-to-ec2-via-browser-ec2-instance-connect)
8. [Step 6 — Install Jenkins on EC2](#step-6--install-jenkins-on-ec2)
9. [Step 7 — Install Required Plugins](#step-7--install-required-plugins)
10. [Step 8 — Connect GitHub to Jenkins](#step-8--connect-github-to-jenkins)
11. [Step 9 — Configure AWS Credentials in Jenkins](#step-9--configure-aws-credentials-in-jenkins)
12. [Step 10 — Create the Sample GitHub Project](#step-10--create-the-sample-github-project)
13. [Step 11 — Create Jenkins Pipeline Job](#step-11--create-jenkins-pipeline-job)
14. [Step 12 — Setup GitHub Webhook (Auto Trigger)](#step-12--setup-github-webhook-auto-trigger)
15. [Step 13 — Run & Verify End to End](#step-13--run--verify-end-to-end)
16. [Troubleshooting](#troubleshooting)
17. [Quick Reference Checklist](#quick-reference-checklist)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS Cloud                            │
│                                                             │
│   ┌──────────────┐    pulls code    ┌──────────────────┐   │
│   │  EC2 Instance │ ◄────────────── │   GitHub Repo    │   │
│   │  (Jenkins)    │                 │  (your project)  │   │
│   │               │                 └──────────────────┘   │
│   │  Port 8080    │                        ▲               │
│   │  (Jenkins UI) │                        │               │
│   └──────┬────────┘              webhook trigger           │
│          │                       (on git push)             │
│          │ uploads artifact                                 │
│          ▼                                                  │
│   ┌──────────────┐                                         │
│   │  S3 Bucket   │                                         │
│   │  (artifacts) │                                         │
│   └──────────────┘                                         │
│                                                             │
│   IAM Role attached to EC2 = S3 access (no keys needed)    │
└─────────────────────────────────────────────────────────────┘

You access Jenkins UI from your browser using EC2 Public IP:8080
You connect to EC2 terminal using EC2 Instance Connect (browser-based, no SSH client needed)
```

---

## What You Will Need

| Requirement | Details |
|---|---|
| AWS Account | With permission to create EC2, S3, IAM |
| GitHub Account | Free account is fine |
| Browser | Chrome or Firefox (to access Jenkins UI and EC2 terminal) |
| Nothing else | No local tools, no SSH client, no AWS CLI locally |

---

## Step 1 — Create IAM Role for EC2

> **Why?** The EC2 instance running Jenkins needs permission to upload files to S3.  
> Attaching an IAM Role is the **secure way** — no AWS keys stored anywhere.

### 1.1 Navigate to IAM

```
AWS Console → Search "IAM" → Roles → Create Role
```

### 1.2 Select Trusted Entity

```
Trusted entity type: AWS Service
Use case: EC2
Click: Next
```

### 1.3 Attach Permission Policy

Search for and select:

```
AmazonS3FullAccess
```

> For production, use a restricted custom policy (shown below). For learning, `AmazonS3FullAccess` is fine.

**Restricted Custom Policy (Production Recommended):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR-BUCKET-NAME",
        "arn:aws:s3:::YOUR-BUCKET-NAME/*"
      ]
    }
  ]
}
```

### 1.4 Name and Create

```
Role name: Jenkins-EC2-S3-Role
Click: Create Role
```

✅ **Done — IAM Role created.**

---

## Step 2 — Create S3 Bucket

> **Why?** This is the storage where Jenkins will upload your build output (artifacts) after every successful build.

### 2.1 Navigate to S3

```
AWS Console → Search "S3" → Create Bucket
```

### 2.2 Bucket Settings

| Field | Value |
|---|---|
| Bucket name | `jenkins-artifacts-yourname` (must be globally unique) |
| AWS Region | `us-east-1` (pick one, use same region for EC2 too) |
| Block all public access | ✅ Keep ON — artifacts should be private |
| Versioning | Enable (optional — lets you keep old builds) |
| Everything else | Leave as default |

### 2.3 Click "Create Bucket"

✅ **Done — note down your bucket name.**

---

## Step 3 — Launch EC2 Instance

### 3.1 Navigate to EC2

```
AWS Console → EC2 → Instances → Launch Instances
```

### 3.2 EC2 Settings

| Field | Value |
|---|---|
| Name | `Jenkins-Server` |
| AMI | **Ubuntu Server 22.04 LTS** |
| Instance type | `t3.medium` (2 vCPU, 4GB RAM — minimum for Jenkins) |
| Key pair | **Proceed without key pair** (we will use EC2 Instance Connect via browser) |
| Security Group | Create new — name it `Jenkins-SG` (configure in next step) |
| Storage | 20 GB, gp3 |
| IAM Instance Profile | `Jenkins-EC2-S3-Role` |

> ✅ **No key pair needed** because we will connect to EC2 directly from the browser using EC2 Instance Connect — no SSH client required on your machine.

> ⚠️ **Important:** Set IAM Instance Profile before launching. Scroll down to **"Advanced details"** → **"IAM instance profile"** → select `Jenkins-EC2-S3-Role`

### 3.3 Click "Launch Instance"

Wait 1-2 minutes for the instance to show **"Running"** status.

---

## Step 4 — Security Group Rules

> **Why?** Security groups are like a firewall. Without correct rules, you cannot reach Jenkins and Jenkins cannot reach GitHub or S3.

### 4.1 Go to Security Group

```
EC2 → Security Groups → Select "Jenkins-SG" → Edit Inbound Rules
```

### 4.2 Inbound Rules

| Type | Protocol | Port | Source | Why Needed |
|---|---|---|---|---|
| SSH | TCP | 22 | `0.0.0.0/0` | EC2 Instance Connect (browser terminal) needs this |
| Custom TCP | TCP | 8080 | `0.0.0.0/0` | Jenkins web UI — access from your browser |

> 💡 **Why SSH is `0.0.0.0/0` here?**  
> EC2 Instance Connect connects from AWS's own IP ranges which change, so open to all is acceptable for this use case. If you later add a real SSH key, restrict SSH to your IP.

> 💡 **Port 8080** is where Jenkins runs by default. Opening it to `0.0.0.0/0` allows you to access Jenkins from your browser.

### 4.3 Outbound Rules

| Type | Protocol | Port | Destination | Why Needed |
|---|---|---|---|---|
| All traffic | All | All | `0.0.0.0/0` | EC2 needs to reach GitHub (to clone repo), S3 (to upload), and internet (to install packages) |

> ✅ Default outbound (allow all) is correct. Do not change it.

### 4.4 Save Rules

Click **"Save rules"**

---

## Step 5 — Connect to EC2 via Browser (EC2 Instance Connect)

> **Why?** Since we are doing everything in the cloud without a local machine, we use EC2 Instance Connect — it opens a terminal directly in your browser. No PuTTY, no SSH key needed.

### 5.1 Open EC2 Instance Connect

```
EC2 → Instances → Select "Jenkins-Server" → Connect (top right button)
→ Select tab: "EC2 Instance Connect"
→ Click: Connect
```

A **black terminal window opens in your browser**. This is your EC2 terminal.

> ✅ All commands from here will be typed in this browser terminal.

---

## Step 6 — Install Jenkins on EC2

> Run all these commands inside the EC2 Instance Connect terminal (browser terminal).

### 6.1 Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

### 6.2 Install Java 17

> Jenkins requires Java to run.

```bash
sudo apt install openjdk-17-jdk -y
```

Verify:

```bash
java -version
```

Expected output:
```
openjdk version "17.x.x"
```

### 6.3 Install Jenkins

```bash
# Add Jenkins GPG key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# Add Jenkins repository
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update and install Jenkins
sudo apt update
sudo apt install jenkins -y
```

### 6.4 Start Jenkins

```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

You should see `active (running)` in green.

### 6.5 Install AWS CLI

> Jenkins will use this to upload files to S3.

```bash
sudo apt install awscli -y
aws --version
```

### 6.6 Install Git

```bash
sudo apt install git -y
git --version
```

### 6.7 Get Jenkins Initial Admin Password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy this password — you need it in the next step.

### 6.8 Open Jenkins in Browser

```
http://YOUR_EC2_PUBLIC_IP:8080
```

> Get the public IP from: EC2 → Instances → your instance → "Public IPv4 address"

**Setup Steps in Jenkins:**
1. Paste the admin password copied above
2. Click **"Install suggested plugins"** — wait for installation
3. Create your admin username and password
4. Click **"Save and Finish"**
5. Click **"Start using Jenkins"**

✅ **Jenkins is running!**

---

## Step 7 — Install Required Plugins

> **Why?** Default Jenkins does not have GitHub integration or S3 upload capability. These plugins add that.

### 7.1 Go to Plugin Manager

```
Jenkins Dashboard → Manage Jenkins → Plugins → Available Plugins
```

### 7.2 Search and Install Each Plugin

Search for each one, check the box, then click **"Install"** at the bottom.

| Plugin Name (Search This) | Why You Need It |
|---|---|
| `Pipeline` | Enables Jenkinsfile-based pipeline jobs |
| `Git` | Lets Jenkins clone your GitHub repo |
| `GitHub` | GitHub integration, webhooks, status updates |
| `GitHub Integration` | Triggers builds automatically on GitHub push |
| `Pipeline: AWS Steps` | Adds `withAWS` and `s3Upload` commands |
| `Credentials Binding` | Safely use secrets inside pipelines |
| `SSH Agent` | For SSH-based GitHub connections (optional) |

### 7.3 Restart Jenkins After Installing

```
http://YOUR_EC2_PUBLIC_IP:8080/safeRestart
```

Wait for Jenkins to come back up, then log in again.

---

## Step 8 — Connect GitHub to Jenkins

> **Why?** Jenkins needs to authenticate with GitHub to clone your private/public repo and to receive webhook triggers.

### 8.1 Create GitHub Personal Access Token (PAT)

> This is done on GitHub, not Jenkins.

```
GitHub → Your Profile (top right) → Settings
→ Developer settings (bottom left)
→ Personal access tokens → Tokens (classic)
→ Generate new token (classic)
```

**Token Settings:**

| Field | Value |
|---|---|
| Note | `Jenkins-Access` |
| Expiration | 90 days or No expiration |
| Scopes to check | `repo` (full), `admin:repo_hook` (for webhooks) |

Click **"Generate token"** → **Copy the token immediately** (you cannot see it again).

---

### 8.2 Add GitHub Token to Jenkins Credentials

```
Jenkins Dashboard → Manage Jenkins → Credentials
→ System → Global credentials (unrestricted)
→ Add Credentials
```

| Field | Value |
|---|---|
| Kind | `Username with password` |
| Username | Your GitHub username |
| Password | Paste the GitHub token you just created |
| ID | `github-credentials` |
| Description | GitHub Personal Access Token |

Click **"Create"**

✅ **Jenkins can now access your GitHub.**

---

## Step 9 — Configure AWS Credentials in Jenkins

> **Why?** Since the EC2 already has the IAM Role attached, the AWS CLI on EC2 automatically uses it. You likely do NOT need to add AWS credentials manually.

### 9.1 Verify IAM Role is Working (on EC2 terminal)

Open EC2 Instance Connect terminal and run:

```bash
aws sts get-caller-identity
```

Expected output:
```json
{
    "UserId": "AROAXXXXXXXXXXXXXXXXX:i-xxxxxxxx",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/Jenkins-EC2-S3-Role/i-xxxxxxxx"
}
```

If you see the role name — ✅ no credentials needed in Jenkins, the pipeline will work automatically.

### 9.2 Test S3 Access from EC2

```bash
aws s3 ls s3://jenkins-artifacts-yourname --region us-east-1
```

If no error → ✅ S3 access confirmed.

---

## Step 10 — Create the Sample GitHub Project

> We will use a simple Node.js project to demonstrate the full pipeline. Fork or create this yourself.

### 10.1 Create a New GitHub Repository

```
GitHub → New Repository
Name: jenkins-s3-demo
Visibility: Public
Initialize with README: ✅ Yes
```

### 10.2 Create Project Files

Inside your GitHub repo, create these files using the GitHub web editor (click "Add file" → "Create new file"):

---

**File 1: `package.json`**

```json
{
  "name": "jenkins-s3-demo",
  "version": "1.0.0",
  "description": "Demo app for Jenkins S3 pipeline",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "test": "echo 'All tests passed!' && exit 0",
    "build": "echo 'Build complete!' && mkdir -p dist && cp index.js dist/"
  }
}
```

---

**File 2: `index.js`**

```javascript
const http = require('http');

const PORT = process.env.PORT || 3000;

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello from Jenkins + S3 Pipeline!\n');
});

server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

**File 3: `Jenkinsfile`** ← Most Important File

```groovy
pipeline {
    agent any

    environment {
        // =============================
        // CHANGE THESE TWO VALUES
        // =============================
        S3_BUCKET   = 'jenkins-artifacts-yourname'   // Your S3 bucket name
        S3_REGION   = 'us-east-1'                    // Your AWS region
        // =============================
        APP_NAME    = 'jenkins-s3-demo'
        BUILD_DIR   = 'dist'
    }

    stages {

        stage('Checkout') {
            steps {
                echo "========== STAGE: Checkout =========="
                echo "Cloning from GitHub..."
                checkout scm
                echo "Branch: ${env.GIT_BRANCH}"
                echo "Commit: ${env.GIT_COMMIT}"
                sh 'ls -la'
            }
        }

        stage('Install Dependencies') {
            steps {
                echo "========== STAGE: Install =========="
                sh 'node --version || echo "Node not installed"'
                // Install Node.js if not present
                sh '''
                    if ! command -v node &> /dev/null; then
                        curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
                        sudo apt install -y nodejs
                    fi
                    node --version
                    npm --version
                '''
                sh 'npm install'
            }
        }

        stage('Test') {
            steps {
                echo "========== STAGE: Test =========="
                sh 'npm test'
            }
        }

        stage('Build') {
            steps {
                echo "========== STAGE: Build =========="
                sh 'npm run build'
                sh 'ls -la dist/'
                echo "Build artifacts created in dist/ folder"
            }
        }

        stage('Package Artifact') {
            steps {
                echo "========== STAGE: Package =========="
                // Create a zip file of the build output
                sh """
                    zip -r ${APP_NAME}-build-${BUILD_NUMBER}.zip ${BUILD_DIR}/
                    ls -lh ${APP_NAME}-build-${BUILD_NUMBER}.zip
                """
            }
        }

        stage('Upload to S3') {
            steps {
                echo "========== STAGE: Upload to S3 =========="
                sh """
                    # Upload the zip artifact
                    aws s3 cp ${APP_NAME}-build-${BUILD_NUMBER}.zip \
                        s3://${S3_BUCKET}/builds/${APP_NAME}/build-${BUILD_NUMBER}/${APP_NAME}-build-${BUILD_NUMBER}.zip \
                        --region ${S3_REGION}

                    # Also upload a latest.zip that always points to newest build
                    aws s3 cp ${APP_NAME}-build-${BUILD_NUMBER}.zip \
                        s3://${S3_BUCKET}/builds/${APP_NAME}/latest.zip \
                        --region ${S3_REGION}
                """
                echo "Artifact uploaded to S3 successfully!"
            }
        }

        stage('Verify Upload') {
            steps {
                echo "========== STAGE: Verify =========="
                sh """
                    echo "Listing S3 bucket contents:"
                    aws s3 ls s3://${S3_BUCKET}/builds/${APP_NAME}/ --region ${S3_REGION} --recursive
                """
            }
        }

    }

    post {
        success {
            echo """
            ============================================
            ✅ BUILD SUCCESS
            App     : ${env.APP_NAME}
            Build # : ${BUILD_NUMBER}
            Branch  : ${env.GIT_BRANCH}
            Commit  : ${env.GIT_COMMIT}
            S3 Path : s3://${S3_BUCKET}/builds/${APP_NAME}/build-${BUILD_NUMBER}/
            ============================================
            """
        }
        failure {
            echo """
            ============================================
            ❌ BUILD FAILED
            Build # : ${BUILD_NUMBER}
            Check the console output above for errors.
            ============================================
            """
        }
        always {
            echo "Cleaning up workspace..."
            sh 'rm -f *.zip || true'
        }
    }
}
```

---

**File 4: `.gitignore`**

```
node_modules/
dist/
*.zip
*.log
```

---

### 10.3 Final GitHub Repo Structure

```
jenkins-s3-demo/
├── Jenkinsfile          ← Pipeline script (Jenkins reads this)
├── package.json         ← Node.js project config
├── index.js             ← Application code
├── .gitignore
└── README.md
```

---

## Step 11 — Create Jenkins Pipeline Job

### 11.1 Create New Pipeline Job

```
Jenkins Dashboard → New Item
→ Item name: jenkins-s3-demo
→ Select: Pipeline
→ Click: OK
```

### 11.2 General Settings

- Check: **"GitHub project"**
- Project URL: `https://github.com/YOUR_USERNAME/jenkins-s3-demo`

### 11.3 Build Triggers

- Check: **"GitHub hook trigger for GITScm polling"**

> This makes Jenkins listen for GitHub webhooks — auto-build on every push.

### 11.4 Pipeline Definition

Scroll down to **Pipeline** section:

| Field | Value |
|---|---|
| Definition | `Pipeline script from SCM` |
| SCM | `Git` |
| Repository URL | `https://github.com/YOUR_USERNAME/jenkins-s3-demo.git` |
| Credentials | Select `github-credentials` (created in Step 8) |
| Branch Specifier | `*/main` |
| Script Path | `Jenkinsfile` |

> **What does "Pipeline script from SCM" mean?**  
> It means Jenkins reads the `Jenkinsfile` directly from your GitHub repo instead of you copying it into Jenkins. This is the recommended approach — your pipeline lives with your code.

### 11.5 Save

Click **"Save"**

---

## Step 12 — Setup GitHub Webhook (Auto Trigger)

> **Why?** Without a webhook, Jenkins will not know when you push code to GitHub. The webhook tells GitHub to notify Jenkins every time there is a push.

### 12.1 Get Jenkins URL

Your Jenkins webhook URL will be:
```
http://YOUR_EC2_PUBLIC_IP:8080/github-webhook/
```

> ⚠️ Make sure port 8080 is open in your Security Group (already done in Step 4).

### 12.2 Add Webhook on GitHub

```
GitHub → Your Repo (jenkins-s3-demo)
→ Settings → Webhooks → Add webhook
```

| Field | Value |
|---|---|
| Payload URL | `http://YOUR_EC2_PUBLIC_IP:8080/github-webhook/` |
| Content type | `application/json` |
| Which events | `Just the push event` |
| Active | ✅ Checked |

Click **"Add webhook"**

### 12.3 Verify Webhook

After saving, GitHub will send a test ping. You should see a green ✅ next to your webhook on the webhooks page.

If you see a red ✗:
- Check EC2 Security Group has port 8080 open to `0.0.0.0/0`
- Check your EC2 Public IP is correct
- Check Jenkins is running: `sudo systemctl status jenkins`

---

## Step 13 — Run & Verify End to End

### 13.1 Trigger First Build Manually

```
Jenkins Dashboard → jenkins-s3-demo → Build Now
```

### 13.2 Watch the Build

```
Click on build #1 → Console Output
```

You will see each stage running:

```
========== STAGE: Checkout ==========
Cloning from GitHub...
...
========== STAGE: Install ==========
...
========== STAGE: Test ==========
All tests passed!
...
========== STAGE: Build ==========
Build complete!
...
========== STAGE: Package ==========
...
========== STAGE: Upload to S3 ==========
upload: jenkins-s3-demo-build-1.zip to s3://jenkins-artifacts-yourname/...
Artifact uploaded to S3 successfully!
...
========== STAGE: Verify ==========
Listing S3 bucket contents:
2024-01-01 10:00:00    1234 builds/jenkins-s3-demo/build-1/jenkins-s3-demo-build-1.zip
2024-01-01 10:00:00    1234 builds/jenkins-s3-demo/latest.zip
...
✅ BUILD SUCCESS
```

### 13.3 Verify File in S3

```
AWS Console → S3 → jenkins-artifacts-yourname
→ builds/ → jenkins-s3-demo/ → build-1/
```

You should see the `.zip` file there.

### 13.4 Test Auto Trigger via GitHub Push

Make any small change in your GitHub repo (edit README.md), commit and push. Jenkins should automatically start a new build within 30 seconds.

---

## Troubleshooting

### ❌ Jenkins page not loading at port 8080

**Check:**
```bash
# In EC2 Instance Connect terminal
sudo systemctl status jenkins
sudo systemctl start jenkins
```
Also verify Security Group has port 8080 open.

---

### ❌ Git clone fails — repository not found

**Check credentials:**
```
Jenkins → Manage Jenkins → Credentials → verify github-credentials exists
```
Also verify repo URL is correct and your GitHub token has `repo` scope.

---

### ❌ S3 upload fails — Access Denied

```bash
# In EC2 Instance Connect terminal, test manually:
aws sts get-caller-identity
aws s3 ls s3://jenkins-artifacts-yourname
```
If this fails, the IAM Role is not attached. Go to:
```
EC2 → Instances → Select instance → Actions → Security → Modify IAM Role → Attach Jenkins-EC2-S3-Role
```

---

### ❌ aws command not found in Jenkins pipeline

```bash
# In EC2 Instance Connect terminal:
which aws
# If blank, install it:
sudo apt install awscli -y

# Check Jenkins can find it:
sudo -u jenkins aws --version
```

If still not found, add this to top of your Jenkinsfile `environment` block:
```groovy
environment {
    PATH = "/usr/local/bin:/usr/bin:/bin:${env.PATH}"
}
```

---

### ❌ npm or node not found

The Jenkinsfile already handles this with an auto-install block. But if it still fails:

```bash
# In EC2 Instance Connect terminal, install manually:
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
node --version
npm --version
```

---

### ❌ Webhook not triggering builds

- Verify port 8080 is open in Security Group to `0.0.0.0/0`
- Verify webhook URL ends with `/github-webhook/` (trailing slash is required)
- Verify Jenkins GitHub plugin is installed
- Check GitHub webhook delivery log (GitHub → Repo Settings → Webhooks → click webhook → Recent Deliveries)

---

### ❌ EC2 Instance Connect not working

- Wait 2-3 minutes after EC2 launches before trying
- Verify Security Group has SSH port 22 open (EC2 Instance Connect uses port 22)
- Use **Ubuntu** AMI — EC2 Instance Connect works best with Ubuntu

---

## Quick Reference Checklist

### AWS Setup
- [ ] IAM Role `Jenkins-EC2-S3-Role` created with S3 permissions
- [ ] S3 bucket created in same region as EC2
- [ ] EC2 launched — Ubuntu 22.04, t3.medium, no key pair needed
- [ ] IAM Role attached to EC2 at launch time
- [ ] Security Group `Jenkins-SG` created with:
  - [ ] Inbound: SSH port 22 open (for EC2 Instance Connect)
  - [ ] Inbound: TCP port 8080 open (for Jenkins UI)
  - [ ] Outbound: All traffic allowed

### EC2 Software Setup (via EC2 Instance Connect)
- [ ] System updated (`apt update && apt upgrade`)
- [ ] Java 17 installed and verified (`java -version`)
- [ ] Jenkins installed and running (`systemctl status jenkins`)
- [ ] AWS CLI installed (`aws --version`)
- [ ] Git installed (`git --version`)
- [ ] IAM Role verified (`aws sts get-caller-identity`)
- [ ] S3 access tested (`aws s3 ls s3://your-bucket`)

### Jenkins Setup
- [ ] Jenkins UI accessible at `http://EC2_IP:8080`
- [ ] Initial setup completed (admin user created)
- [ ] Plugins installed: Pipeline, Git, GitHub, GitHub Integration, Pipeline AWS Steps, Credentials Binding
- [ ] Jenkins restarted after plugins
- [ ] GitHub credentials added (username + PAT token)

### GitHub Setup
- [ ] GitHub repo created with: `Jenkinsfile`, `package.json`, `index.js`, `.gitignore`
- [ ] Webhook added pointing to `http://EC2_IP:8080/github-webhook/`
- [ ] Webhook shows green checkmark in GitHub

### Jenkins Job Setup
- [ ] Pipeline job created: `jenkins-s3-demo`
- [ ] GitHub project URL set
- [ ] GitHub hook trigger enabled
- [ ] Pipeline from SCM configured (pointing to your GitHub repo)
- [ ] Jenkinsfile has correct S3 bucket name and region

### Verification
- [ ] Manual build triggered and succeeded
- [ ] Console output shows all stages green
- [ ] Artifact visible in S3 bucket
- [ ] Git push triggered automatic build

---

## What Each Component Does — Summary

| Component | Role in This Setup |
|---|---|
| EC2 Instance | The server that runs Jenkins 24/7 in the cloud |
| EC2 Instance Connect | Browser-based terminal — connect to EC2 without any local tools |
| IAM Role | Gives EC2 permission to write to S3 — no keys needed |
| S3 Bucket | Storage for build artifacts (zip files) after each build |
| Security Group | Firewall — allows only necessary traffic in and out |
| Jenkins | The automation engine — runs your pipeline on every code push |
| GitHub Repo | Where your code and Jenkinsfile live |
| GitHub Webhook | Notifies Jenkins when you push code — triggers automatic builds |
| Jenkinsfile | The pipeline script that defines all the build/test/upload steps |
| GitHub PAT Token | Allows Jenkins to securely clone your private or public GitHub repo |
| AWS CLI | Command-line tool Jenkins uses to upload files to S3 |
| Pipeline AWS Steps Plugin | Adds AWS commands as Jenkins pipeline steps (alternative to CLI) |

---

*Documentation version: 2.0 | Cloud-only setup | Jenkins LTS + GitHub + AWS S3*
