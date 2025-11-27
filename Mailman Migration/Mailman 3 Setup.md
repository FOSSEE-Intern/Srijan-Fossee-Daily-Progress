This guide assumes you are starting with a minimal AlmaLinux 10 installation and will be running system commands as a user with `sudo` privileges.

### 1. VM Setup and Network Configuration

### Download iso files

- - Almalinux 10: [https://repo.almalinux.org/almalinux/10/isos/x86_64/AlmaLinux-10.0-x86_64-minimal.iso](https://repo.almalinux.org/almalinux/10/isos/x86_64/AlmaLinux-10.0-x86_64-minimal.iso)

### Configure Virtual Machines

- Adapter 1: NAT
- Adapter 2: Host-only Adapter

### Configure Network Interfaces

After installation, run these commands on both VMs:

```sh
# List network connections
sudo nmcli con show

# Configure NAT adapter (enp0s3)
sudo nmcli con modify enp0s3 ipv4.dns "8.8.8.8"
sudo nmcli con up enp0s3

# Configure Host-only adapter (enp0s8)
sudo nmcli con add type ethernet con-name enp0s8 ifname enp0s8
sudo nmcli con modify enp0s8 ipv4.addresses "192.168.56.X/24" # Replace X with any IP (eg 192.168.56.15 for CentOS, 192.168.56.16 for AlmaLinux)
sudo nmcli con modify enp0s8 ipv4.method manual
sudo nmcli con modify enp0s8 ipv4.gateway ""
sudo nmcli con up enp0s8
```

### 2. Install System Dependencies

We use `dnf` (or `yum`) for package management and install PostgreSQL, Postfix, development tools, and utility libraries.

```bash
# Update system and install basic development tools (including GCC)
sudo dnf update -y
sudo dnf groupinstall "Development Tools" -y

# Install Python 3.9+ (Python 3.12 is likely default/available) and development headers
sudo dnf install python3 python3-devel -y

# Install PostgreSQL (RHEL/Alma uses 'postgresql-server' for the daemon)
sudo dnf install postgresql-server postgresql-devel -y

# Install Mail Transfer Agent (MTA)
sudo dnf install postfix -y

# Install utility packages (gettext for compilemessages, sassc for web UI)
sudo dnf install epel-release -y
sudo dnf install gettext sassc lynx -y
```

### 3. Initialize and Enable PostgreSQL

Unlike Debian, PostgreSQL often requires manual initialization on RHEL-based systems.

```bash
# Initialize the database (run once)
sudo postgresql-setup --initdb

# Enable and start the PostgreSQL service
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

### 4. Configure Postfix MTA

Postfix is installed but needs to be configured and started.

```bash
# Enable and start Postfix service
sudo systemctl enable postfix
sudo systemctl start postfix

# Configure Postfix to relay mail to Mailman Core
# Edit /etc/postfix/main.cf (as root) and ADD/APPEND these lines:
sudo vi /etc/postfix/main.cf
# ... (add these lines)
unknown_local_recipient_reject_code = 550
owner_request_special = no
transport_maps =
    hash:/opt/mailman/mm/var/data/postfix_lmtp
local_recipient_maps =
    $alias_maps
    hash:/opt/mailman/mm/var/data/postfix_lmtp
relay_domains =
    hash:/opt/mailman/mm/var/data/postfix_domains
```

### 5. Setup Database (PostgreSQL)

You will create the `mailman` user and the two necessary databases. **Replace `[MYPASSWORD]` with a strong secret password.**

```bash
sudo -u postgres psql

postgres=# create user mailman with encrypted password '[MYPASSWORD]';
postgres=# create database mailman owner mailman;
postgres=# create database mailmanweb owner mailman;
postgres=# grant all privileges on database mailman to mailman;
postgres=# grant all privileges on database mailmanweb to mailman;
postgres=# \q
```

---

## II. User and Virtual Environment Setup

### 6. Setup Mailman User and Directories

Create the dedicated user and necessary file paths.

```bash
# Create user and set permissions
sudo useradd -m -d /opt/mailman -s /bin/bash mailman
sudo chown mailman:mailman /opt/mailman
sudo chmod 755 /opt/mailman

# Switch to mailman user (for all subsequent Python commands)
sudo su mailman
mkdir /opt/mailman/mm --mode 755

# Create web directories (correctly owned by mailman user)
mkdir -p /opt/mailman/web/logs
mkdir -p /opt/mailman/web/static
```

### 7. Setup and Activate Virtualenv


```bash
cd /opt/mailman
python3 -m venv venv
source /opt/mailman/venv/bin/activate
# Your prompt should now start with (venv)
```

### 8. Install Python Packages

Install Mailman Core, the Web components, and the WSGI server.

```python
# inside venv environment
pip install setuptools wheel mailman psycopg2-binary
pip install mailman-web mailman-hyperkitty
pip install uwsgi
```

---

## III. Configuration Files

*Run these commands as a root user.*
### 9. Create Configuration Directory

```bash
sudo mkdir -p /etc/mailman3
```

### 10. Configure Mailman Core (`mailman.cfg`)

Create `/etc/mailman3/mailman.cfg` (as root) and substitute `[MYPASSWORD]` and `[YOUR_EMAIL]`.

```bash
sudo vi /etc/mailman3/mailman.cfg
# ... (file contents)
[paths.here]
var_dir: /opt/mailman/mm/var
[mailman]
layout: here
site_owner: [YOUR_EMAIL] 
[database]
class: mailman.database.postgresql.PostgreSQLDatabase
url: postgresql://mailman:[MYPASSWORD]@localhost/mailman
[archiver.prototype]
enable: yes
[archiver.hyperkitty]
class: mailman_hyperkitty.Archiver
enable: yes
configuration: /etc/mailman3/mailman-hyperkitty.cfg
[shell]
history_file: $var_dir/history.py
[mta]
verp_confirmations: yes
verp_personalized_deliveries: yes
verp_delivery_interval: 1
```

### 11. Configure Hyperkitty Archiver (`mailman-hyperkitty.cfg`)

Create `/etc/mailman3/mailman-hyperkitty.cfg` (as root) and substitute `[YOUR_HYPERKITTY_KEY]`.

```bash
sudo nano /etc/mailman3/mailman-hyperkitty.cfg
# ... (file contents)
[general]
base_url: http://127.0.0.1:8000/archives/
api_key: [YOUR_HYPERKITTY_KEY] 
```

### 12. Configure Mailman Web (`settings.py`)

Create `/etc/mailman3/settings.py` (as root). **Substitute all five placeholders** (especially `ALLOWED_HOSTS` with your Host-Only IP).

```python
# /etc/mailman3/settings.py
from mailman_web.settings.base import *
from mailman_web.settings.mailman import *

ADMINS = (
    ('Mailman Suite Admin', '[YOUR_EMAIL]'),
)

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'mailmanweb',
        'USER': 'mailman',
        'PASSWORD': '[MYPASSWORD]',  
        'HOST': 'localhost',
        'PORT': '5432',
    }}

STATIC_ROOT = '/opt/mailman/web/static'
COMPRESS_ENABLED = True
LOGGING['handlers']['file']['filename'] = '/opt/mailman/web/logs/mailmanweb.log'

ALLOWED_HOSTS = [
    "localhost",
    "127.0.0.1",
    "[YOUR_HOST_ONLY_IP]", 
]

SITE_ID = 1
SECRET_KEY = '[MY_DJANGO_SECRET_KEY]' 
MAILMAN_ARCHIVER_KEY = '[YOUR_HYPERKITTY_KEY]' 
DEFAULT_FROM_EMAIL = 'admin@[YOUR_DOMAIN]'
SERVER_EMAIL = 'admin@[YOUR_DOMAIN]'
```

---
## Change PostgreSQL Authentication Method

You need to edit PostgreSQL's Host-Based Authentication (HBA) configuration file, `pg_hba.conf`, and reload the service.

### Step 1: Locate and Edit `pg_hba.conf`

The location of this file can vary, but it's typically under `/var/lib/pgsql/data/` or `/var/lib/pgsql/1x/data/`.

1. **Stop the PostgreSQL service:**

```bash
 sudo systemctl stop postgresql
```

2. **Find the configuration file (as root/sudo):**

```bash
# Common location on RHEL/Alma
sudo nano /var/lib/pgsql/data/pg_hba.conf
```
### Step 2: Modify Authentication Entries

Inside `pg_hba.conf`, find the lines that deal with `host` (network) and `local` (Unix socket) connections. They likely look like this:

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             all                                     peer
host    all             all             127.0.0.1/32            ident
host    all             all             ::1/128                 ident
```

**Change the `METHOD` from `ident` or `peer` to `md5`** for the local loopback addresses (`127.0.0.1` and `::1`) and for local sockets.

*Modify them to look like this:*

![[Pasted image 20251127190018.png]]
### Step 3: Start and Reload PostgreSQL

1. **Start the service:**

```bash
sudo systemctl start postgresql
```

---
## IV. Web UI Setup and Service Configuration

### 13. Run Web Management Commands

Run these commands as the `mailman` user with `venv` activated.

```bash
# inside venv environment
mailman-web migrate
mailman-web collectstatic
mailman-web compress
mailman-web compilemessages
```

### 14. Configure uWSGI

Create the configuration file for the WSGI server at `/etc/mailman3/uwsgi.ini` (as root).

```bash
sudo vi /etc/mailman3/uwsgi.ini
# ... (file contents)
[uwsgi]
http-socket = 0.0.0.0:8000
virtualenv = /opt/mailman/venv/
module=mailman_web.wsgi:application
env = PYTHONPATH=/etc/mailman3/
env = DJANGO_SETTINGS_MODULE=settings
master = true
processes = 2
threads = 2
attach-daemon = /opt/mailman/venv/bin/mailman-web qcluster
req-logger = file:/opt/mailman/web/logs/uwsgi.log
logger = qcluster file:/opt/mailman/web/logs/uwsgi-qcluster.log
log-route = qcluster uwsgi-daemons
logger = file:/opt/mailman/web/logs/uwsgi-error.log
```

### 15. Create Systemd Service Files

Create `/etc/systemd/system/mailman3.service` and `/etc/systemd/system/mailmanweb.service` as root.

**`mailman3.service`:** (Note: Uses `postgresql` service name)

```toml
[Unit]
Description=GNU Mailing List Manager
After=syslog.target network.target postgresql.service
[Service]
Type=forking
PIDFile=/opt/mailman/mm/var/master.pid
User=mailman
Group=mailman
Environment="MAILMAN_CONFIG_FILE=/etc/mailman3/mailman.cfg"
ExecStart=/opt/mailman/venv/bin/mailman start
ExecReload=/opt/mailman/venv/bin/mailman restart
ExecStop=/opt/mailman/venv/bin/mailman stop
[Install]
WantedBy=multi-user.target
```

**`mailmanweb.service`:**

```toml
[Unit]
Description=GNU Mailman Web UI
After=syslog.target network.target postgresql.service mailman3.service
[Service]
Environment="PYTHONPATH=/etc/mailman3/"
User=mailman
Group=mailman
ExecStart=/opt/mailman/venv/bin/uwsgi --ini /etc/mailman3/uwsgi.ini
KillSignal=SIGINT
[Install]
WantedBy=multi-user.target
```

### 16. Start and Enable Services

```bash
sudo systemctl daemon-reload
sudo systemctl enable mailman3.service
sudo systemctl enable mailmanweb.service
sudo systemctl start mailman3
sudo systemctl start mailmanweb
sudo systemctl status mailman3
sudo systemctl status mailmanweb
```

### 17. Setup Cron Jobs

Setup cron jobs for the `mailman` user (includes both Core and Web tasks):


```bash
sudo -u mailman crontab -e
```

**Add the following lines:**

```
@daily /opt/mailman/venv/bin/mailman digests --periodic
@daily /opt/mailman/venv/bin/mailman notify

# Web UI/Django QCluster jobs
* * * * * /opt/mailman/venv/bin/mailman-web runjobs minutely
0,15,30,45 * * * * /opt/mailman/venv/bin/mailman-web runjobs quarter_hourly
@hourly            /opt/mailman/venv/bin/mailman-web runjobs hourly
@daily             /opt/mailman/venv/bin/mailman-web runjobs daily
@weekly            /opt/mailman/venv/bin/mailman-web runjobs weekly
@monthly           /opt/mailman/venv/bin/mailman-web runjobs monthly
@yearly            /opt/mailman/venv/bin/mailman-web runjobs yearly
```

---

## V. Web Server and Final Access

### 18. Install and Configure Nginx

Install Nginx and configure it to reverse proxy requests from port 80 to the uWSGI server on port 8000.

```bash
sudo dnf install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Comment out **`server`** block the Nginx default configuration file (path may vary, often `/etc/nginx/nginx.conf` or `/etc/nginx/conf.d/default.conf`) 

```nginx
server {
   listen 80;
   server_name _; 

   location /static/ {
        alias /opt/mailman/web/static/;
   }

   location / {
           proxy_pass http://127.0.0.1:8000;
           proxy_set_header Host $host;
           proxy_set_header X-Forwarded-For $remote_addr;
   }
}
```

Create a file `/etc/nginx/conf.d/mailman.conf` and insert this lines in file:

```nginx
server {
   listen 80;
   listen [::]:80;
   server_name [YOUR_HOST_ONLY_IP]; # Your VM's Host-Only IP

   # Serve static files directly from the file system
   location /static/ {
        alias /opt/mailman/web/static/;
   }

   # Reverse proxy all other requests to the Mailman uWSGI/Gunicorn server
   location / {
           proxy_pass http://127.0.0.1:8000;
           proxy_set_header Host $host;
           proxy_set_header X-Forwarded-For $remote_addr;
   }
}
```

Restart Nginx:

Bash

```bash
sudo systemctl restart nginx
```

### 19. Create Superuser

Create the administrative user for the web interface.

```bash
$ sudo su mailman
$ source /opt/mailman/venv/bin/activate
(venv)$ mailman-web createsuperuser
# Follow the prompts to set up your username and password.
```

### 20. Firewall Configuration

AlmaLinux uses `firewalld`. You must open port 80 (or 443 for HTTPS) for external access from your host machine.

```bash
sudo firewall-cmd --zone=public --add-service=http --permanent
# If using HTTPS:
# $ sudo firewall-cmd --zone=public --add-service=https --permanent
sudo firewall-cmd --reload
```