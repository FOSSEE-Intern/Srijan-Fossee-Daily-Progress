### The Architecture: "The Intelligent Funnel"

In a professional setup, you **never** send 100% of traffic to a heavy AI model (Transformers/BERT). It is too expensive (CPU/Latency) and wasteful. You want a funnel:

1. **Level 1 (Postfix):** Accepts connection.
2. **Level 2 (Rspamd Standard):** fast checks (DNS blocklists, Regex, DKIM/DMARC). Rejects obvious spam immediately.
3. **Level 3 (The AI Oracle):** If Level 2 is "unsure" (gray area), it sends the email text to your custom Hugging Face Microservice for a "second opinion."
---

### Phase 1: Environment & Postfix Setup

_Goal: Get the mail server running and ready for filtering._

**1. Update & Install Prerequisites** Switch to root (or use sudo) and install necessary tools.

```bash
sudo dnf update -y
sudo dnf install -y epel-release postfix git python3-pip python3-devel gcc
```

**2. Configure Postfix** Edit the main configuration file:

```bash
sudo vi /etc/postfix/main.cf
```

Add/Modify these lines to ensure Postfix listens on all interfaces and is ready for the Milter (Rspamd) later:

```toml
# Basic Networking
inet_interfaces = all
myhostname = mail.localdomain  # Change to your VM hostname

# Milter Configuration (Rspamd will run on port 11332)
smtpd_milters = inet:127.0.0.1:11332
non_smtpd_milters = inet:127.0.0.1:11332
milter_protocol = 6
milter_mail_macros = i {mail_addr} {client_addr} {client_name} {auth_authen}
milter_default_action = accept
```

Start Postfix:

```bash
sudo systemctl enable --now postfix
```

![Alt Text](../images/Screenshot 2026-01-29 150248.png)
---
### Phase 2: Install & Configure Rspamd

### Step 1: Install Podman

AlmaLinux 10 prefers Podman over Docker (commands are compatible).

```bash
sudo dnf install -y podman
```

### Step 2: Create the Data Directories

We need folders on your VM to store the Rspamd configurations so they persist if the container restarts.

```bash
sudo mkdir -p /etc/rspamd/local.d
sudo mkdir -p /var/lib/rspamd
```

### Step 3: Start Rspamd Container

Run this exact block. It pulls the official Rspamd image and maps the ports to your localhost so Postfix can see it.

```bash
sudo podman run -d \
  --name rspamd \
  --restart always \
  -p 127.0.0.1:11332:11332 \
  -p 127.0.0.1:11333:11333 \
  -p 0.0.0.0:11334:11334 \
  -v /etc/rspamd/local.d:/etc/rspamd/local.d:Z \
  -v /var/lib/rspamd:/var/lib/rspamd:Z \
  -v /etc/rspamd/rspamd.local.lua:/etc/rspamd/rspamd.local.lua:Z \
  rspamd/rspamd:latest
```

- **`127.0.0.1:11332`**: The port Postfix will talk to (Milter).
- **`0.0.0.0:11334`**: The Web Interface (so you can access it from your browser).
- **`:Z`**: A special flag for SELinux (mandatory on AlmaLinux) to allow the container to read the files.
### Step 4: Verify it is Running

Check if the container is up and listening.

```bash
sudo podman ps
# You should see 'rspamd/rspamd:latest' with Status 'Up'
```

Then check if the port is open on your VM:

```bash
sudo ss -tulpn | grep 11332
```

_Expected Output:_ You should see a process listening on `127.0.0.1:11332`.
### Step 5: Verify Postfix Connection

Now that Rspamd is listening on port 11332, restart Postfix to make it reconnect.

```bash
sudo systemctl restart postfix
```

Check the logs again to ensure the "Connection Refused" error is gone:

```bash
sudo tail -n 20 /var/log/maillog
```

![Alt Text](../images/Screenshot%202026-01-29%20150304.png)

---
### Phase 3: The AI Microservice (Python & Hugging Face)

We will now set up the Python environment on your **Host VM** (not the container). This service will download the AI model and listen for requests.
#### Step 1: Install Python Dependencies

AlmaLinux 10 usually comes with Python 3.11 or 3.12. We need `pip` and the development tools.

```bash
sudo dnf install -y python3-pip python3-devel gcc
```

#### Step 2: Set up the Project Environment

We will keep this clean by using a virtual environment (`venv`).

```bash
# Create directory
sudo mkdir -p /opt/ai-spam-filter
sudo chown $USER:$USER /opt/ai-spam-filter
cd /opt/ai-spam-filter

# Create virtual environment
python3 -m venv venv

# Activate it
source venv/bin/activate

# Install the heavy lifters (This will take 2-5 minutes depending on internet speed)

pip install --upgrade pip

pip install torch --index-url https://download.pytorch.org/whl/cpu
pip install fastapi uvicorn transformers
```

#### Step 3: Create the AI Application

Now, create the actual Python script that acts as the "Brain."

Create the file:

```bash
vi /opt/ai-spam-filter/main.py
```

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from transformers import pipeline
import uvicorn
import os

# Initialize FastAPI
app = FastAPI()

# ---------------------------------------------------------
# MODEL CONFIGURATION
# ---------------------------------------------------------
MODEL_NAME = "dima806/email-spam-detection-distilbert"

print(f"INITIALIZING: Loading AI Model {MODEL_NAME}...")

# device=-1 enforces CPU usage explicitly.
# This prevents the library from checking for NVIDIA drivers on startup.
spam_classifier = pipeline("text-classification", model=MODEL_NAME, device=-1)

print("READY: Model loaded and running on CPU.")

class EmailPayload(BaseModel):
    text: str

@app.post("/scan")
async def scan_email(payload: EmailPayload):
    # 1. TRUNCATE: Limit to 2000 chars to save CPU cycles
    scan_text = payload.text[:2000]

    try:
        # 2. INFERENCE
        result = spam_classifier(scan_text)[0]
        
        # 3. LOGIC: Check if label is SPAM
        # The model returns "SPAM" or "LABEL_1" depending on version
        label = result['label'].upper()
        is_spam = "SPAM" in label or "LABEL_1" in label
        
        return {
            "is_spam": is_spam,
            "confidence": float(result['score']),
            "char_count": len(scan_text)
        }

    except Exception as e:
        print(f"ERROR: {str(e)}")
        raise HTTPException(status_code=500, detail="AI Processing Failed")

if __name__ == "__main__":
    # Listen on localhost port 8000
    uvicorn.run(app, host="127.0.0.1", port=8000)
```

#### Step 4: Test the AI (Manual Run)

Before we automate it, let's run it manually to make sure it downloads the model and works.

1. **Start the server:**
```bash
python main.py
```
	_Wait until you see: `Application startup complete`._
2. **Open a NEW terminal window** (keep the python one running) and test it with `curl`:
```bash
curl -X POST "http://127.0.0.1:8000/scan" \
	 -H "Content-Type: application/json" \
	 -d '{"text": "Congratulations! You have won a free iPhone. Click here."}'
```
**Expected Output:**
You should see a JSON response saying `is_spam: true`.

![[Screenshot 2026-01-29 150324.png]]

---

The manual test passed, and the model is now cached on your disk. This means subsequent starts will be instant.
### Step 1: Create the Systemd Service File

We need to tell Linux how to manage your Python script.

Create the service file:
```bash
sudo vi /etc/systemd/system/ai-spam.service
```

Paste the following configuration. (Note: I have updated the paths to match your exact setup):

```toml
[Unit]
Description=AI Spam Filter Microservice
After=network.target

[Service]
# Run as root to ensure no permission issues during the demo
User=root
WorkingDirectory=/opt/ai-spam-filter

# The command to start the app. 
# We point directly to the python executable inside the 'venv' folder.
# We add '--workers 2' to limit CPU usage as discussed.
ExecStart=/opt/ai-spam-filter/venv/bin/python -m uvicorn main:app --host 127.0.0.1 --port 8000 --workers 2

# Restart automatically if it crashes
Restart=always
RestartSec=5

# Output logs to syslog (viewable via journalctl)
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=ai-spam

[Install]
WantedBy=multi-user.target
```

### Step 2: Enable and Start the Service

Now, reload the system manager to pick up the new file, and start your AI engine.

```bash
# Reload systemd configuration
sudo systemctl daemon-reload

# Enable it to start on boot
sudo systemctl enable ai-spam

# Start it right now
sudo systemctl start ai-spam
```

### Step 3: Verify It Is "Alive"

Check the status to ensure it's running green and happy.

```bash
sudo systemctl status ai-spam
```

_You should see: `Active: active (running)`_

### Step 4: The Final "Black Box" Test

Before we glue this to Rspamd, let's verify the API is listening in the background. Run this `curl` command from your terminal:

```bash
curl -X POST "http://127.0.0.1:8000/scan" \
     -H "Content-Type: application/json" \
     -d '{"text": "URGENT: You have won a lottery! Send money to claim prize."}'
```

**What to look for:**

If you get a JSON response like `{"is_spam":true, "confidence":0.99...}`, then **Phase 3 is 100% complete**.

---

## Connecting the "Brain" (Host) to the "Body" (Rspamd Container).

### The Networking Challenge

You have a networking mismatch right now:

- **The Brain (Python):** Listens on `127.0.0.1` (Localhost of the VM).
- **The Body (Rspamd):** Lives inside a container.
- **The Problem:** If Rspamd tries to call `127.0.0.1`, it will look inside _itself_ (the container), not at your VM.

We must make the Python API accessible from the "outside" (the container network) and tell Rspamd exactly where to look.

---
### Step 1: Open the AI Service to the Network

Currently, your AI service only accepts connections from `localhost`. We need it to listen on `0.0.0.0` (All interfaces) so the container can reach it.

1. Edit the service file:
```bash
sudo vi /etc/systemd/system/ai-spam.service
```

2. Find the `ExecStart` line. Change `--host 127.0.0.1` to `--host 0.0.0.0`:
```toml
ExecStart=/opt/ai-spam-filter/venv/bin/python -m uvicorn main:app --host 0.0.0.0 --port 8000 --workers 2
```

3. Reload and restart:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ai-spam
```
    _(Wait ~10 seconds for the model to reload into RAM)._

---

### Step 2: Find Your Host IP Address

We need the IP address that the container can use to talk to the Host.
```bash
ip addr
```

- Use your VM's main IP address (e.g., `192.168.X.X`).
- **Copy this IP.** You will need it in the next step.

---
### Step 3: Create the Rspamd "Glue" Script

We will now write the Lua script that sends emails from Rspamd to your Python API.

1. Create the file (this folder is mapped to your container):
```bash
sudo vi /etc/rspamd/rspamd.local.lua
```

2. Paste the code below. **IMPORTANT: Replace `YOUR_HOST_IP_HERE` with the IP you copied in Step 2.**
```lua
local rspamd_logger = require "rspamd_logger"
local rspamd_http = require "rspamd_http"
local ucl = require "ucl"

local function ai_scan_callback(task)
  -- 1. Optimization: Skip if score is already high
  local score = task:get_metric_score('default')[1]
  if score > 15 then return end 

  local content = task:get_content()
  if not content or #content < 10 then return end

  -- 2. Prepare the JSON body using UCL (Cleaner & Safer)
  local request_data = {
    text = content
  }
  -- Convert table to JSON string automatically
  local request_body = ucl.to_format(request_data, 'json-compact')

  -- 3. Define Callback
  local function http_callback(err_code, code, body, headers)
    if err_code then
      rspamd_logger.errx(task, "AI API Error: %s", err_code)
      return
    end

    local parser = ucl.parser()
    local res, err = parser:parse_string(body)
    
    if res then
      local obj = parser:get_object()
      -- THE DECISION LOGIC
      if obj.is_spam and obj.confidence > 0.90 then
        task:insert_result('AI_SPAM', 20.0, 'confidence: ' .. obj.confidence)
        rspamd_logger.infox(task, "AI DETECTED SPAM! Confidence: %s", obj.confidence)
      end
    end
  end

  -- 4. SEND REQUEST
  -- REPLACE '10.88.0.1' WITH YOUR ACTUAL HOST IP
  rspamd_http.request({
    url = 'http://10.88.0.1:8000/scan',
    body = request_body,
    task = task,
    method = 'post',
    mime_type = 'application/json',
    timeout = 2.0,
    callback = http_callback
  })
end

rspamd_config:register_symbol({
  name = 'AI_SPAM_SCAN',
  type = 'callback',
  callback = ai_scan_callback
})
```

---
### Step 4: Activate the Script

We need to register this new check in Rspamd's configuration.

1. Create/Edit `rspamd.conf.local`:
```bash
sudo vi /etc/rspamd/rspamd.conf.local
```

2. Add this block:
```lua
rspamd_config:register_symbol({
  name = 'AI_SPAM_SCAN',
  type = 'callback',
  callback = rspamd_config.AI_SPAM_SCAN.callback
})
```

3. Create the Metrics Configuration File** Run this command to create the file specifically for scores:
```bash
sudo vi /etc/rspamd/local.d/metrics.conf
```

```bash
symbol "AI_SPAM" {
weight = 20.0;
description = "AI Detected Spam via Python API";
}
```

4. Restart the Rspamd Container to apply changes:
```bash
sudo podman restart rspamd
```
---

==***If you recieve a similar kind of error.***==
###  Permission Denied (`stats.ucl`)

**The Error:** `cannot open for writing controller stats from /var/lib/rspamd/stats.ucl... Permission denied`

You created the directory `/var/lib/rspamd` on your Host VM using `sudo`, so it is owned by **root**. However, inside the container, Rspamd drops its privileges to a standard user (usually named `rspamd`, UID 100) for security. This "internal user" tries to write to the folder but gets blocked by the "root" ownership on the host.

**The Fix:** Since this is a demo environment, we will make that specific folder writable by everyone so the container user can save data.

Run this on your **Host VM**:

```bash
sudo chmod -R 777 /var/lib/rspamd
```

_Restart the container to verify the error stops:_

```bash
sudo podman restart rspamd
```
---

### Step 5: The Grand Finale (End-to-End Test)

Everything is connected. Let's watch it work live.

1. **Terminal 1 (Watch the Logs):**
We want to see Rspamd shouting that it found spam.
```bash
sudo podman logs -f rspamd
```

2. **Terminal 2 (Send the Attack):**
Open a new terminal window and simulate a spammer.
```bash
telnet localhost 25
```

```plaintext
EHLO spammer.com
MAIL FROM: <badguy@spammer.com>
RCPT TO: <pmaurya@localhost>
DATA
Subject: You won a prize!

URGENT: You have won a lottery! Send money to claim prize immediately.
.
QUIT
```

**Check Terminal 1:**

You should see a log line appear:

`... (default) ... AI DETECTED SPAM! Confidence: 0.99...`

**Check Terminal 2:**

Postfix should reject you with: `554 5.7.1 Spam message rejected`

![[Screenshot 2026-01-29 150331 1.png]]

---
> "We used Podman because the required shared libraries for Rspamd aren't stable on AlmaLinux 10 yet. Instead of hacking the OS dependencies, I containerized Rspamd. This isolates the service and ensures reliability regardless of OS updates."




