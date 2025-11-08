## 1. Install required dependencies

Update and configure firewall

```sh
sudo dnf update -y

# allow port 8081
sudo firewall-cmd --permanent --add-port=8080/tcp

#reload the firewall
sudo firewall-cmd --reload

# check if its success full or not
sudo firewall-cmd --list-ports
```

Install MySQL

```sh
sudo dnf install mysql8.4-server
sudo systemctl start mysqld
sudo systemctl enable mysqld
sudo mysql_secure_installation
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

```sh
sudo systemctl restart mysqld
# Try logging into mysql
mysql -u root -p
```

## 2. Install Keycloak

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

```shell
# Login to MySQl
sudo mysql -u root -p

CREATE DATABASE keycloakdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'keycloakuser'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON keycloakdb.* TO 'keycloakuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Download MySQL JDBC Driver and add it into keycloak

```sh
cd /opt/keycloak/providers
sudo wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.4.0/mysql-connector-j-8.4.0.jar
sudo chown keycloak:keycloak mysql-connector-j-8.4.0.jar
# SELinux file context for provider JAR
sudo chcon -u system_u -t usr_t /opt/keycloak/providers/mysql-connector-j-8.4.0.jar
```

Edit `/opt/keycloak/conf/keycloak.conf`:

```shell
sudo vi /opt/keycloak/conf/keycloak.conf

# Update/add these lines

#Configure keycloak to use MySQL instead of H2
db=mysql
db-username=keycloakuser
db-password=your_password
db-url=jdbc:mysql://localhost:3306/keycloakdb

# Proxy & Hostname settings
proxy=edge
proxy-headers=xforwarded

hostname=your_ip_addr
http-enabled=true
https-enabled=false
http-port=8080
```

Create a temporary admin:

```sh
cd /opt/keycloak
sudo -u keycloak ./bin/kc.sh build
export P=your_password
sudo --preserve-env=P ./bin/kc.sh bootstrap-admin user --username admin --password:env P
```

> **Note**: This admin is temporary. After logging into the Admin Console, create a permanent admin user with the `admin` role, then remove the bootstrap user.

Build and verify

```shell
cd /opt/keycloak
sudo -u keycloak ./bin/kc.sh start --optimized
```

Create the service file `/etc/systemd/system/keycloak.service`:

```shell
[Unit]
Description=Keycloak Authorization Server
After=network.target

[Service]
Type=idle
User=keycloak
Group=keycloak
WorkingDirectory=/opt/keycloak
ExecStart=/opt/keycloak/bin/kc.sh start --optimized
LimitNOFILE=102400
LimitNPROC=102400
TimeoutStartSec=600
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

Set correct owner and selinux file attributes.

```shell
sudo chcon -u system_u -t systemd_unit_file_t /etc/systemd/system/keycloak.service
sudo chown -R keycloak:keycloak /opt/keycloak
sudo chmod -R 755 /opt/keycloak

# Enable keycloak daemon service.
sudo systemctl daemon-reload
sudo systemctl enable --now keycloak
sudo systemctl status keycloak
```

## 3. Configure Keycloak client

### Drupal: Get Redirect URL

- Open your Drupal site.
- Login and navigate to Configuration > Web Services > OpenID Connect.
- Under Enabled OpenID Connect clients check the Keycloak
- Copy Redirect url given

### Keycloak: Replace temporary admin with newly cleared permanent admin.**(Important)**

+ Login into keycloak 
+ Navigate to Users
+ Click on add users
+ Give a username, email, firstName, secondName
+ Click on email verified
+ Click on create user
+ Once user is created click on Role Mapping and give it admin access
+ Now once done, delete the temp user.
### Create keycloak client

- Open your Keycloak console at `http://{keycloak_domain}:8080`.
- Login and navigate to Clients > Create Client.
- General Settings: Set Client type to `OpenID Connect` and Client ID to `openplc`. Click Next.
- Capability config: Ensure Client authentication is On.
- Login settings: Paste the URL into Valid redirect URIs: `{redirect_url}`
  Hit Save.
- Navigate to the Credentials tab and Copy the Client secret.