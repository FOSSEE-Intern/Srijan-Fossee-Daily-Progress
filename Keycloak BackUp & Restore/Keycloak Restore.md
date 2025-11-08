Now you should have a new blank VM -- lets restore keycloak

# 1. Install required dependencies
---

```
sudo dnf update -y

# allow port 8081
sudo firewall-cmd --permanent --add-port=8080/tcp

#reload the firewall
sudo firewall-cmd --reload

# check if its success full or not
sudo firewall-cmd --list-ports
```

Install MySQL

```bash
sudo dnf install mysql8.4-server
sudo systemctl start mysqld
sudo systemctl enable mysqld
sudo mysql_secure_installation
```
```
Press y|Y for Yes, any other key for No: n
Please set the password for root here.

New password:

Re-enter new password:

Remove anonymous users? (Press y|Y for Yes, any other key for No) : y
Success.

Disallow root login remotely? (Press y|Y for Yes, any other key for No) : y

Remove test database and access to it? (Press y|Y for Yes, any other key for No) : y

Reload privilege tables now? (Press y|Y for Yes, any other key for No) : y

All done!
```

Restart mysqld

```bash
sudo systemctl restart mysqld
# Try logging into mysql
mysql -u root -p
```

# 2. Install Keycloak
---

```bash
# Install java
sudo dnf install java-21-openjdk-devel -y
cd /opt
# Download the latest stable Keycloak (check keycloak.org/downloads for new versions)
sudo wget https://github.com/keycloak/keycloak/releases/download/26.4.0/keycloak-26.4.0.zip
sudo dnf install unzip
sudo unzip keycloak-26.4.0.zip
sudo mv keycloak-26.4.0 keycloak

# Create a dedicated user for Keycloak
sudo groupadd keycloak
sudo useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak
sudo chown -R keycloak:keycloak /opt/keycloak

# Adjust SELinux context so that keycloak files can be executed without being blocked
sudo chcon -R -u system_u -t usr_t /opt/keycloak
```

Create database and user for keycloak:

```bash
# Login to MySQl
sudo mysql -u root -p

CREATE DATABASE keycloakdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'keycloakuser'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON keycloakdb.* TO 'keycloakuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Download MySQL JDBC Driver and add it into keycloak

```bash
cd /opt/keycloak/providers
sudo wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.4.0/mysql-connector-j-8.4.0.jar
sudo chown keycloak:keycloak mysql-connector-j-8.4.0.jar
# SELinux file context for provider JAR
sudo chcon -u system_u -t usr_t /opt/keycloak/providers/mysql-connector-j-8.4.0.jar
```

# 3. Copy Backup Files to the New VM

Copy both files from your previous machine:

```bash
scp -r /var/backups/keycloak/2025-11-09_01-20-00 <username>@<new-vm-ip>:/home/<username>/
```

# 4. Restore your database dump

Now restore your SQL backup file:
```bash
mysql -u keycloakuser -p'password' keycloakdb < /home/<username>/2025-11-09_01-20-00/keycloak-db.sql
```

You’ll see no output if it’s successful 

# 5. Restore Keycloak Configuration

Now unpack your configuration backup:
```
sudo dnf install tar
sudo tar -xzf /home/<username>/2025-11-09_01-20-00/keycloak-config.tar.gz -C /
```
That restores your `/opt/keycloak/conf/` directory (hostname, ports, etc.)
```bash
cat /opt/keycloak/conf/keycloak.conf
```
Confirm your `/opt/keycloak/conf/keycloak.conf` the same old configuration, also adjust the hostname.

# 6. Start Keycloak

```
cd /opt/keycloak
sudo -u keycloak ./bin/kc.sh build
sudo -u keycloak ./bin/kc.sh start --optimized
```

Keycloak should start running.

Navigate to your browser and hit 
```
http://<your-new-vm-ip>:8080
```

If you can log in with your old admin credentials and see your old realms → **Disaster Recovery is Successful!**

