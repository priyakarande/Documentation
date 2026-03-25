# EasyCRUD — Jenkins CI/CD Pipeline with S3 Artifact Storage

> **Project:** Student Registration App (Spring Boot + React + MySQL)  
> **Repo:** https://github.com/Priyaks/EasyCRUD-Updated  
> **What this does:** Jenkins pulls code from GitHub, builds the Spring Boot backend using Maven, and uploads the `.jar` file to an S3 bucket for storage.

---

## What We Are Doing and Why

This project has two parts — a React frontend and a Spring Boot backend. Right now we are only automating the **backend build and artifact storage**.

Here is the simple flow:

```
You push code to GitHub
        ↓
Jenkins on EC2 detects the push
        ↓
Jenkins pulls the latest code
        ↓
Maven builds the backend → produces a .jar file
        ↓
Jenkins uploads the .jar to S3 bucket
        ↓
Your artifact is safely stored and versioned
```

The `.jar` file that Maven creates from your `pom.xml` is the artifact. It is the compiled, runnable version of your Spring Boot app. We store it in S3 so we always have a copy of every build, and we can redeploy any previous version anytime without rebuilding.

---

## Project Structure (For Reference)

```
EASYCRUD-UPDATED/
├── compose.yml
├── README.md
├── backend/
│   ├── pom.xml                          ← Maven reads this to build
│   ├── dockerfile
│   ├── src/
│   │   └── main/
│   │       ├── java/com/student/registration/
│   │       │   ├── StudentRegistrationBackendApplication.java
│   │       │   ├── config/WebConfig.java
│   │       │   ├── controller/UserController.java
│   │       │   ├── model/User.java
│   │       │   └── repository/UserRepository.java
│   │       └── resources/
│   │           └── application.properties  ← DB config lives here
│   └── src/test/
└── frontend/
    ├── package.json
    ├── vite.config.js
    ├── .env                              ← API URL config
    └── src/
        ├── App.jsx
        ├── api/userService.js
        └── components/
```

After Maven builds, it creates:

```
backend/target/student-registration-backend-0.0.1-SNAPSHOT.jar   ← this is your artifact
```

This `.jar` is what we upload to S3.

---

## Requirements Before You Start

| Requirement | Details |
|---|---|
| AWS Account | Needs permission to create EC2, S3, IAM |
| GitHub Account | Repo is already public — good |
| Browser | Chrome or Firefox |
| Nothing else | Everything runs in AWS cloud, no local tools needed |

---

## Step 1 — Create IAM Role for EC2

> The Jenkins EC2 needs permission to upload files to S3. We do this by attaching an IAM Role — no AWS keys stored anywhere.

### Go to IAM

```
AWS Console → IAM → Roles → Create Role
```

### Settings

```
Trusted entity type: AWS Service
Use case: EC2
Click: Next
```

Search and attach this policy:

```
AmazonS3FullAccess
```

Name the role:

```
Role name: Jenkins-EC2-S3-Role
Click: Create Role
```

✅ Done.

---

## Step 2 — Create S3 Bucket

> This is where your built `.jar` files will be stored after every Jenkins build.

```
AWS Console → S3 → Create Bucket
```

### Settings

| Field | Value |
|---|---|
| Bucket name | `oncdec-online-b35-my-buxx` (this is the bucket name already in your pipeline) |
| Region | `ap-south-1` (Mumbai — matches your pipeline) |
| Block all public access | Keep ON |
| Everything else | Leave as default |

Click **Create Bucket**

✅ Done.

> If you want a different bucket name, create it here and update `bucket` in the Jenkinsfile later.

---

## Step 3 — Launch EC2 Instance (Jenkins Server)

```
AWS Console → EC2 → Instances → Launch Instances
```

### Settings

| Field | Value |
|---|---|
| Name | `Jenkins-Server` |
| AMI | Ubuntu Server 22.04 LTS |
| Instance type | `t3.medium` (2 vCPU, 4 GB RAM — Jenkins needs at least this) |
| Key pair | Proceed without key pair (we use browser terminal) |
| Security Group | Create new — name it `Jenkins-SG` |
| Storage | 20 GB, gp3 |
| IAM Instance Profile | `Jenkins-EC2-S3-Role` |

> Important: To attach the IAM Role, scroll down on the launch page to **Advanced details** → **IAM instance profile** → select `Jenkins-EC2-S3-Role`

Click **Launch Instance** and wait for status to show **Running**.

---

## Step 4 — Security Group Rules

> Security group controls what traffic is allowed in and out of your EC2.

### Inbound Rules

| Port | Protocol | Source | Why |
|---|---|---|---|
| 22 | TCP | `0.0.0.0/0` | EC2 Instance Connect (browser terminal) needs this |
| 8080 | TCP | `0.0.0.0/0` | Jenkins runs on port 8080 — you open it in browser |

### Outbound Rules

| Port | Protocol | Destination | Why |
|---|---|---|---|
| All | All | `0.0.0.0/0` | EC2 needs to reach GitHub to clone code, S3 to upload, and internet to install packages |

> Do not change outbound rules. All outbound traffic must be allowed.

---

## Step 5 — Connect to EC2 from Browser

> No SSH client or PuTTY needed. AWS lets you open a terminal directly in your browser.

```
EC2 → Instances → Select Jenkins-Server → Connect (top right)
→ Tab: EC2 Instance Connect
→ Click: Connect
```

A black terminal window opens in your browser. All commands from this point go here.

---

## Step 6 — Install Everything on EC2

Run these commands one by one inside the browser terminal.

### Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

### Install Java 17

> Spring Boot and Jenkins both need Java.

```bash
sudo apt install openjdk-17-jdk -y
java -version
```

You should see something like `openjdk version "17.x.x"`

### Install Maven

> Maven is what Jenkins uses to build your Spring Boot backend and produce the `.jar` file.

```bash
sudo apt install maven -y
mvn -version
```

### Install Jenkins

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y

sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

You should see `active (running)` in the output.

### Install AWS CLI

> Jenkins uses this to upload the `.jar` to S3.

```bash
sudo apt install awscli -y
aws --version
```

### Install Git

```bash
sudo apt install git -y
git --version
```

### Verify IAM Role is Working

> Since we attached the IAM Role to EC2, AWS credentials are automatic. Verify it works:

```bash
aws sts get-caller-identity
```

You should see your AWS account ID and the role name `Jenkins-EC2-S3-Role` in the output. If you see this, Jenkins can upload to S3 without any credentials stored anywhere.

Test S3 access directly:

```bash
aws s3 ls s3://oncdec-online-b35-my-buxx --region ap-south-1
```

No error means S3 access is working.

---

## Step 7 — Set Up Jenkins

### Open Jenkins in Your Browser

```
http://YOUR_EC2_PUBLIC_IP:8080
```

Get the EC2 public IP from: EC2 → Instances → your instance → Public IPv4 address

### Get the Initial Admin Password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the output and paste it in the Jenkins browser page.

### Complete Setup

1. Click **Install suggested plugins** — wait for it to finish
2. Create your admin username and password
3. Click **Save and Finish**
4. Click **Start using Jenkins**

---

## Step 8 — Install Required Jenkins Plugins

```
Jenkins Dashboard → Manage Jenkins → Plugins → Available Plugins
```

Search and install each of these:

| Plugin | Why You Need It |
|---|---|
| `Pipeline` | Lets Jenkins run a Jenkinsfile pipeline |
| `Git` | Lets Jenkins pull code from GitHub |
| `GitHub` | GitHub connection and webhook support |
| `Pipeline: AWS Steps` | Adds `withAWS` and `s3Upload` commands used in your pipeline |
| `Credentials Binding` | Lets Jenkins safely use stored credentials inside pipeline |

After installing all plugins:

```
http://YOUR_EC2_PUBLIC_IP:8080/safeRestart
```

Wait for Jenkins to restart, then log back in.

---

## Step 9 — Store AWS Credentials in Jenkins

> Even though the IAM Role handles S3 access automatically, your pipeline uses `withAWS(credentials: 'creds', ...)` which means Jenkins looks for a stored credential with the ID `creds`. We need to create this.

> You have two options:

### Option A — Use IAM Role (Cleaner, Recommended)

Since IAM Role is already attached, you can create an empty/dummy credential entry with ID `creds` or update the pipeline to remove `withAWS` and use the CLI directly. See the updated Jenkinsfile in Step 11.

### Option B — Store AWS Keys (What Your Current Pipeline Expects)

If you want to keep `withAWS(credentials: 'creds')` exactly as is:

```
Jenkins → Manage Jenkins → Credentials
→ System → Global credentials → Add Credentials
```

| Field | Value |
|---|---|
| Kind | AWS Credentials |
| ID | `creds` |
| Description | AWS credentials for S3 |
| Access Key ID | Your AWS Access Key |
| Secret Access Key | Your AWS Secret Key |

Click **Create**

> To get AWS Access Keys: AWS Console → IAM → Users → Your user → Security credentials → Create access key

---

## Step 10 — Add GitHub to Jenkins

> Jenkins needs to pull code from your GitHub repo. Since your repo is public, Jenkins can clone it without credentials. But adding a token is good practice for webhooks and rate limits.

### Create GitHub Personal Access Token

```
GitHub → Profile → Settings → Developer settings
→ Personal access tokens → Tokens (classic) → Generate new token
```

| Field | Value |
|---|---|
| Note | Jenkins-EasyCRUD |
| Scopes | `repo`, `admin:repo_hook` |

Copy the token.

### Add to Jenkins

```
Jenkins → Manage Jenkins → Credentials
→ System → Global credentials → Add Credentials
```

| Field | Value |
|---|---|
| Kind | Username with password |
| Username | Your GitHub username |
| Password | Paste the token |
| ID | `github-credentials` |

Click **Create**

---

## Step 11 — Add Jenkinsfile to Your Project

> The Jenkinsfile tells Jenkins exactly what to do — pull code, build, upload to S3. It should live in the root of your GitHub repo.

Create a file called `Jenkinsfile` in the root of `EASYCRUD-UPDATED/` and paste this:

```groovy
pipeline {
    agent any

    environment {
        // Change these if your bucket name or region is different
        S3_BUCKET = 'oncdec-online-b35-my-buxx'
        S3_REGION = 'ap-south-1'
        JAR_PATH  = 'backend/target/student-registration-backend-0.0.1-SNAPSHOT.jar'
    }

    stages {

        stage('Pull Code') {
            steps {
                echo 'Pulling latest code from GitHub...'
                git branch: 'main', url: 'https://github.com/Priyaks/EasyCRUD-Updated.git'
                echo 'Code pulled successfully'
            }
        }

        stage('Build Backend') {
            steps {
                echo 'Building Spring Boot backend with Maven...'
                sh '''
                    cd backend
                    mvn clean package -DskipTests
                '''
                echo 'Build complete. JAR file created inside backend/target/'
            }
        }

        stage('Upload JAR to S3') {
            steps {
                echo 'Uploading artifact to S3...'
                withAWS(credentials: 'creds', region: "${S3_REGION}") {
                    s3Upload(
                        acl: 'Private',
                        bucket: "${S3_BUCKET}",
                        file: "${JAR_PATH}",
                        path: "builds/build-${BUILD_NUMBER}/student-registration-backend.jar"
                    )
                }
                echo "JAR uploaded to s3://${S3_BUCKET}/builds/build-${BUILD_NUMBER}/"
            }
        }

        stage('Verify Upload') {
            steps {
                echo 'Checking S3 to confirm upload...'
                sh """
                    aws s3 ls s3://${S3_BUCKET}/builds/ --region ${S3_REGION} --recursive
                """
            }
        }

    }

    post {
        success {
            echo """
            ✅ BUILD SUCCESS
            Build Number : ${BUILD_NUMBER}
            Artifact     : student-registration-backend.jar
            Stored at    : s3://${S3_BUCKET}/builds/build-${BUILD_NUMBER}/
            """
        }
        failure {
            echo """
            ❌ BUILD FAILED
            Build Number : ${BUILD_NUMBER}
            Check the console output above to find which stage failed.
            """
        }
        always {
            echo 'Pipeline finished.'
        }
    }
}
```

### What Each Stage Does

| Stage | What Happens |
|---|---|
| `Pull Code` | Jenkins clones your GitHub repo to its workspace on EC2 |
| `Build Backend` | Goes into `backend/` folder, runs `mvn clean package`, produces the `.jar` |
| `Upload JAR to S3` | Takes the `.jar` from `backend/target/` and uploads it to your S3 bucket |
| `Verify Upload` | Lists the S3 bucket contents so you can confirm the file is there |

> Note: `-DskipTests` in the Maven command skips running tests during build. Remove it if you want tests to run before building.

---

## Step 12 — Create Jenkins Pipeline Job

```
Jenkins Dashboard → New Item
```

| Field | Value |
|---|---|
| Item name | `EasyCRUD-Backend-Pipeline` |
| Type | Pipeline |

Click **OK**

### Configure the Job

**General section:**
- Check: **GitHub project**
- Project URL: `https://github.com/Priyaks/EasyCRUD-Updated`

**Build Triggers section:**
- Check: **GitHub hook trigger for GITScm polling**

**Pipeline section:**

| Field | Value |
|---|---|
| Definition | Pipeline script from SCM |
| SCM | Git |
| Repository URL | `https://github.com/Priyaks/EasyCRUD-Updated.git` |
| Credentials | Select `github-credentials` |
| Branch | `*/main` |
| Script Path | `Jenkinsfile` |

Click **Save**

---

## Step 13 — Add GitHub Webhook

> This makes Jenkins automatically start a build every time you push code to GitHub. Without this you have to click "Build Now" manually every time.

### Webhook URL

```
http://YOUR_EC2_PUBLIC_IP:8080/github-webhook/
```

### Add to GitHub

```
GitHub → EasyCRUD-Updated repo → Settings → Webhooks → Add webhook
```

| Field | Value |
|---|---|
| Payload URL | `http://YOUR_EC2_PUBLIC_IP:8080/github-webhook/` |
| Content type | `application/json` |
| Which events | Just the push event |
| Active | Checked |

Click **Add webhook**

GitHub will send a test ping. You should see a green checkmark next to the webhook. If you see red, check that port 8080 is open in your Security Group.

---

## Step 14 — Run the Pipeline

### First Build — Manual

```
Jenkins Dashboard → EasyCRUD-Backend-Pipeline → Build Now
```

### Watch the Console Output

```
Click on build #1 → Console Output
```

What you should see:

```
Pulling latest code from GitHub...
Code pulled successfully

Building Spring Boot backend with Maven...
[INFO] BUILD SUCCESS
Build complete. JAR file created inside backend/target/

Uploading artifact to S3...
JAR uploaded to s3://oncdec-online-b35-my-buxx/builds/build-1/

Checking S3 to confirm upload...
2024-xx-xx  builds/build-1/student-registration-backend.jar

✅ BUILD SUCCESS
Build Number : 1
Artifact     : student-registration-backend.jar
Stored at    : s3://oncdec-online-b35-my-buxx/builds/build-1/
```

### Verify in AWS S3 Console

```
AWS Console → S3 → oncdec-online-b35-my-buxx → builds → build-1
```

You should see `student-registration-backend.jar` there.

### Test Auto Trigger

Make any small change in your GitHub repo (even edit README.md), commit and push. Jenkins should automatically start build #2 within 30 seconds.

---

## Troubleshooting

### Jenkins page not loading

```bash
sudo systemctl status jenkins
sudo systemctl start jenkins
```

Also check that port 8080 is open in the Security Group.

---

### Maven build fails — `pom.xml` not found

This usually means the working directory is wrong. Your `pom.xml` is inside the `backend/` folder, not the root. The Jenkinsfile handles this with `cd backend` before running `mvn`. If you see this error, double check the build stage in your Jenkinsfile has `cd backend` before `mvn clean package`.

---

### S3 Upload fails — `withAWS` step not found

The `Pipeline: AWS Steps` plugin is not installed. Go to:

```
Manage Jenkins → Plugins → Available → search "Pipeline AWS Steps" → Install
```

Restart Jenkins after installing.

---

### S3 Upload fails — Access Denied

Check the IAM Role is attached to the EC2 instance:

```bash
aws sts get-caller-identity
```

If this fails, go to EC2 → Instances → Select your instance → Actions → Security → Modify IAM Role → attach `Jenkins-EC2-S3-Role`

If using `withAWS(credentials: 'creds')`, make sure the `creds` credential ID exists in Jenkins with valid AWS keys.

---

### `mvn` command not found

```bash
sudo apt install maven -y
mvn -version
```

---

### Webhook not triggering builds

- Check port 8080 is open in Security Group to `0.0.0.0/0`
- Webhook URL must end with `/github-webhook/` — the trailing slash is required
- Check GitHub → Repo Settings → Webhooks → Recent Deliveries for error details

---

## Quick Checklist

### AWS Setup
- [ ] IAM Role `Jenkins-EC2-S3-Role` created with `AmazonS3FullAccess`
- [ ] S3 bucket `oncdec-online-b35-my-buxx` created in `ap-south-1`
- [ ] EC2 launched — Ubuntu 22.04, t3.medium
- [ ] IAM Role attached to EC2
- [ ] Security Group allows port 22 and 8080 inbound, all outbound

### EC2 Setup
- [ ] Java 17 installed — `java -version`
- [ ] Maven installed — `mvn -version`
- [ ] Jenkins installed and running — `systemctl status jenkins`
- [ ] AWS CLI installed — `aws --version`
- [ ] Git installed — `git --version`
- [ ] IAM Role verified — `aws sts get-caller-identity`
- [ ] S3 access tested — `aws s3 ls s3://oncdec-online-b35-my-buxx`

### Jenkins Setup
- [ ] Jenkins UI accessible at `http://EC2_IP:8080`
- [ ] Admin user created
- [ ] Plugins installed: Pipeline, Git, GitHub, Pipeline AWS Steps, Credentials Binding
- [ ] Jenkins restarted after plugins
- [ ] `creds` AWS credentials added (if using `withAWS` with stored keys)
- [ ] `github-credentials` added with GitHub PAT

### GitHub + Pipeline
- [ ] `Jenkinsfile` added to root of `EASYCRUD-UPDATED` repo and pushed
- [ ] Jenkins job `EasyCRUD-Backend-Pipeline` created
- [ ] Job configured to use Pipeline from SCM pointing to your repo
- [ ] Webhook added in GitHub pointing to `http://EC2_IP:8080/github-webhook/`
- [ ] Webhook shows green checkmark in GitHub

### Verification
- [ ] Manual build triggered and succeeded
- [ ] JAR file visible in S3 under `builds/build-1/`
- [ ] Git push triggered automatic build

---

## What Happens to the Frontend

The React frontend (`frontend/`) is not part of this pipeline yet. Right now it runs separately. When you are ready to automate the frontend as well, the flow would be:

```bash
cd frontend
npm install
npm run build
# uploads dist/ folder to S3 as a static website
aws s3 sync dist/ s3://your-frontend-bucket/ --region ap-south-1
```

But that is a separate step for later. The current pipeline focuses only on building and storing the Spring Boot backend artifact.

---

*Project: EasyCRUD Student Registration | Stack: Spring Boot + React + MySQL | Pipeline: Jenkins + S3*
