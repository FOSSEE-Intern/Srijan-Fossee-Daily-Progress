## **Phase 0: Configure Jenkins Server (Server A)**

### **0.1 Install Software** 

```bash
sudo dnf update
sudo dnf install git
```
### **0.2 Install Jenkins (LTS Release)**

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo yum upgrade
# Add required dependencies for the jenkins package
sudo yum install fontconfig java-21-openjdk
sudo yum install jenkins
sudo systemctl daemon-reload
```
### **0.3 Start Jenkins**

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

### **0.4 Install Node.js 24** 

```bash
curl -fsSL https://rpm.nodesource.com/setup_24.x | sudo bash -
sudo dnf install -y nodejs
node -v
npm -v
```

You should see something like:

```
v24.x.x
```

---
### **0.5 Install pnpm** 

#### **Option 1: Using Corepack (Recommended)**

Node.js 24 includes **corepack**, which is the preferred way.

```bash
sudo corepack enable
corepack prepare pnpm@latest --activate
```

---

#### **Option 2: Install pnpm via npm (Fallback)**
Use this only if Corepack is disabled for some reason.

```bash
sudo npm install -g pnpm
```

Verify:

```bash
pnpm -v
```

---

## 🔍 Final Verification

```bash
node -v
npm -v
pnpm -v
```

---
## **Phase 1: Configure the Web Server (Server B)**

We need to prepare the server to receive files and serve them securely.

### **1.1 Install Software**

```bash
sudo dnf install nginx rsync policycoreutils-python-utils -y
# (Use 'apt install' if on Ubuntu/Debian)
```

### **1.2 Create the Deployment User**

For security, we never deploy as `root`.

```bash
sudo adduser deployer
sudo passwd deployer # Set a strong password (only needed for initial testing)
```

### **1.3 Create Directory Structure**

```bash
# Create the main folder
sudo mkdir -p /var/www/my-astro-site/releases

# Assign ownership to the deployer
sudo chown -R deployer:deployer /var/www/my-astro-site

# Set permissions so Nginx (others) can read/execute
sudo chmod -R 755 /var/www/my-astro-site
```

### **1.4 Configure Nginx**

Create the configuration file.

```bash
sudo vi /etc/nginx/conf.d/astro-site.conf
```

**Paste the following:**
```nginx
server {
    listen 80;
    server_name YOUR_SERVER_IP_OR_DOMAIN;

    # Point to the 'current' symlink
    # NOTE: If your build creates a 'dist' folder inside, use /current/dist
    root /var/www/my-astro-site/current; 
    
    index index.html;

    # Compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # SPA Routing
    location / {
        try_files $uri $uri/ = 404;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, no-transform";
    }
}
```

**Verify and Restart:**

```bash
sudo nginx -t
sudo systemctl enable --now nginx
```

### **1.5 Configure SELinux (Crucial for RedHat/CentOS)**

Allow Nginx to read files created by the deployer.

```bash
# 1. Allow Nginx to act as a web server
sudo setsebool -P httpd_can_network_connect 1

# 2. Set the context permanently for the directory
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/my-astro-site(/.*)?"

# 3. Apply the context
sudo restorecon -Rv /var/www/my-astro-site
```

---

## **Phase 2: SSH Trust (Credential Setup)**

We need Jenkins to log in to the Web Server without a password.
### **2.1 Generate Keys (On Jenkins Server)**

Log in to your Jenkins server terminal as your normal user.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/jenkins_astro_key
# Press Enter for NO passphrase
```

- **Private Key:** `~/.ssh/jenkins_astro_key`
- **Public Key:** `~/.ssh/jenkins_astro_key.pub`
### **2.2 Add Public Key to Web Server**

*Copy the content of jenkins_astro_key.pub.*

On Web Server:

```bash
sudo su - deployer
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "PASTE_PUBLIC_KEY_CONTENT_HERE" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
exit
```

### **2.3 Add Private Key to Jenkins UI**

1. **Copy Private Key:** `cat ~/.ssh/jenkins_astro_key`
2. Go to **Jenkins Dashboard** → **Manage Jenkins** → **Credentials** → **Global** → **Add Credentials**.
3. **Kind:** SSH Username with private key.
4. **ID:** `astro-deployer-key` (Save this ID).
5. **Username:** `deployer`.
6. **Private Key:** Paste the key content directly.
7. **Passphrase:** Empty.

---

## **Phase 3: Provide Access for the Private Git Repo**

### **3.1: Create a Personal Access Token (PAT) on GitHub**

Since you cannot use your GitHub login password, you need to generate a token that acts as a password.

1. Log in to your GitHub account
2. Go to **Settings** (top right profile icon) > **Developer settings** (at the very bottom of the left sidebar).
3. Select **Personal access tokens** > **Tokens (classic)**.
4. Click **Generate new token (classic)**
5. **Note:** Give it a name (e.g., "Jenkins-CI").
6. **Scopes:** Check the `repo` box (this gives full control of private repositories).
7. Click **Generate token**.
8. **Copy this token immediately.** You won't see it again.
### **3.2: Add the Token to Jenkins**

1. Go to your Jenkins Dashboard.
2. Navigate to **Manage Jenkins** > **Credentials** > **System** > **Global credentials (unrestricted)**.
3. Click **+ Add Credentials**.
4. Fill in the details:
    - **Kind:** `Username with password` (Yes, select this one).
    - **Scope:** Global.
    - **Username:** `CoolSrj06` (Your GitHub username).
    - **Password:** **[PASTE YOUR TOKEN HERE]** (Do not use your GitHub password).
    - **ID:** `github-coolsrj-token` (Or any name you can recognize).
    - **Description:** "GitHub PAT for CoolSrj06".
5. Click **Create**.
---
## **Phase 4: Configure Jenkins Environment**

### **4.1 Install NodeJS Plugin**

- Go to **Manage Jenkins** → **Plugins** → **Available Plugins**.
- Search for and install the following:
    - **NodeJS** (for building the app).
    - **SSH Agent** (CRITICAL: for the `sshagent` step in the pipeline).
- **Restart Jenkins** after installation.

### **4.2 Configure Global Tools**

- **Manage Jenkins** → **Tools**.
- Scroll to **NodeJS**.
- Click **Add NodeJS**.
    - **Name:** `node-24` (Remember this name).
    - **Version:** Select NodeJS 24.x.
    - **Global npm packages to install:** `pnpm` (Just the name, not the command).
- Click **Save**.

---

## **Phase 5: The Pipeline (Jenkinsfile)**

Use this script in your pipeline definition.

**Important:** Update the `REMOTE_HOST` and `GIT_URL` variables below.

```groovy
pipeline {
    agent any

    environment {
        // CONFIGURATION
        REMOTE_USER = 'deployer'
        REMOTE_HOST = 'YOUR_WEB_SERVER_IP' // <--- REPLACE WITH SERVER B IP
        DEPLOY_PATH = '/var/www/my-astro-site'
        NEW_RELEASE = "${DEPLOY_PATH}/releases/${BUILD_ID}"
        SSH_CRED_ID = 'astro-deployer-key' // The ID we created in Phase 2
        
        GIT_CREDENTIALS_ID = 'github-coolsrj-token' // The ID we created in Phase 3
        GIT_URL = 'https://github.com/CoolSrj06/IT.Web.Page.git' // <--- REPLACE THIS
        GIT_BRANCH = 'master'
    }

    tools {
        nodejs 'node-24' 
    }

    stages {
        stage('Checkout Code') {
            steps {
                git credentialsId: GIT_CREDENTIALS_ID, 
                branch: GIT_BRANCH, 
                url: GIT_URL
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'pnpm --version'
                // --frozen-lockfile is pnpm's version of 'npm ci'
                // It ensures you install exactly what is in pnpm-lock.yaml
                sh 'pnpm install --frozen-lockfile' 
            }
        }

        stage('Build Site') {
            steps {
                sh 'pnpm run build' 
            }
        }

        stage('Deploy to Server B') {
            steps {
                sshagent([SSH_CRED_ID]) {
                    script {
                        // 1. Create directory for this release
                        sh "ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} 'mkdir -p ${NEW_RELEASE}'"
                        
                        // 2. Upload dist folder
                        // Astro defaults to 'dist'. Ensure your pnpm build outputs there.
                        sh "rsync -avz --delete dist/ ${REMOTE_USER}@${REMOTE_HOST}:${NEW_RELEASE}/"
                        
                        // 3. Atomic Switch: Update the 'current' symlink
                        sh "ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} 'ln -sfn ${NEW_RELEASE} ${DEPLOY_PATH}/current'"
                    }
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                 sshagent([SSH_CRED_ID]) {
                    // Keep last 5 builds
                    sh "ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} 'cd ${DEPLOY_PATH}/releases && ls -t | tail -n +6 | xargs -r rm -rf'"
                 }
            }
        }
    }
}
```

---

## **Phase 6: Create and Automate Job**

### **6.1 Create the Job**

- **New Item** → Name: `Astro-Production` → **Select Pipeline**.
- Click **OK**.
- Scroll down to **Build Triggers**.
- Check the box: **GitHub hook trigger for GITScm polling**.
- Scroll down to **Pipeline** (Definition).
- Select **Pipeline script** and paste the code from **Phase 5**.
- Click **Save**.

### **6.2 Configure GitHub Webhook**

This step ensures that whenever you push code to GitHub, Jenkins automatically starts the build.

1. Go to your **GitHub Repository** page.
2. Click **Settings** (top navigation bar).
3. On the left sidebar, click **Webhooks**.
4. Click **Add webhook**.
5. **Payload URL:**
```
http://YOUR_JENKINS_SERVER_IP:8080/github-webhook/
```
*Make sure to include the trailing slash*

6. **Content type:** Select `application/json`.
7. **Secret:** Leave empty (unless configured in Jenkins).
8. **Which events would you like to trigger this webhook?** Select "Just the push event".
9. Click **Add webhook**.

_Note: If your Jenkins server is on a private network (localhost/AWS private IP), GitHub cannot reach it. You may need to use a tool like Ngrok or ensure your Security Group allows port 8080 from the internet._

![[Pasted image 20251230140556.png]]
## **Phase 7: Final Verification**

1. **Push a change:** Edit a file in your project (e.g., `README.md`) and push it to the `master` branch on GitHub.
    
2. **Check Jenkins:** Go to your Jenkins Dashboard. You should see the `Astro-Production` job start automatically within a few seconds.
    
3. **Check Website:** Visit your Web Server IP. The changes should be live.

![[Pasted image 20251230132913.png]]

## **Troubleshooting Checklist**

|**Issue**|**Symptom**|**Fix**|
|---|---|---|
|**Jenkins Build Fail**|`pnpm: command not found`|Check Global Tools Config (Phase 4.2). Ensure "pnpm" is in "Global npm packages".|
|**Jenkins Build Fail**|`sshagent: command not found`|Install the **SSH Agent** plugin via Manage Jenkins → Plugins.|
|**Jenkins Build Fail**|`Host key verification failed`|Ensure `StrictHostKeyChecking=no` is in the ssh command in Jenkinsfile.|
|**No Auto Trigger**|Job doesn't start on push|Check GitHub Webhook delivery status (should be a green check). Check if Jenkins allows port 8080 traffic.|
|**Website Error**|**403 Forbidden**|Run `sudo chmod -R 755 /var/www/my-astro-site` on Web Server.|
|**Website Error**|**403 / 500 Internal Error**|Run `getenforce`. If "Enforcing", run the `chcon` and `semanage` commands from Phase 1.5.|
|**Website Error**|**Welcome to Nginx**|Nginx config `root` path is wrong. Check if `index.html` is in `current` or `current/dist`.|
