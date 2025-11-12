+ For using Keycloak v21.0.0 we need to install OpenJdk-17 as version 21 and version less than 17 are not supported. 
+ Regarding the operating system you can use either Rocky Linux 9 or 10, I am as of now working on Rocky Linux 10.


## Install Open JDK 17
### Step 1 — Download and Extract the archive

```bash
wget https://download.oracle.com/java/17/archive/jdk-17.0.12_linux-x64_bin.tar.gz

sudo mkdir -p /usr/lib/jvm
sudo tar -xvzf jdk-17.0.12_linux-x64_bin.tar.gz -C /usr/lib/jvm/
```

This will extract into:

```
/usr/lib/jvm/jdk-17.0.12/
```

---

### Step 2 — Set environment variables

Create a file `/etc/profile.d/jdk17.sh` to make Java 17 available system-wide:

```bash
sudo vi /etc/profile.d/jdk17.sh
```

Add these lines:

```bash
export JAVA_HOME=/usr/lib/jvm/jdk-17.0.12
export PATH=$JAVA_HOME/bin:$PATH
```

Save and exit.

Now reload the environment:

```bash
source /etc/profile.d/jdk17.sh
```

---

### Step 3 — Register with `alternatives`

To integrate with the system’s `java` command chooser:

```bash
sudo alternatives --install /usr/bin/java java /usr/lib/jvm/jdk-17.0.12/bin/java 2
sudo alternatives --install /usr/bin/javac javac /usr/lib/jvm/jdk-17.0.12/bin/javac 2
```

Then select it:

```bash
sudo alternatives --config java
sudo alternatives --config javac
```

Choose the entry pointing to `/usr/lib/jvm/jdk-17.0.12/bin/java`.

---

### Step 4 — Verify the installation

Check Java version:

```bash
java -version
```

✅ You should see something like:

```
java version "17.0.12" 2024-07-16 LTS
Java(TM) SE Runtime Environment (build 17.0.12+8-LTS-123)
Java HotSpot(TM) 64-Bit Server VM (build 17.0.12+8-LTS-123, mixed mode, sharing)
```

---

Now you should have a new blank VM -- lets restore keycloak

## Install required dependencies
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
sudo dnf install mysql-server
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

## Install Keycloak
---

```bash
cd /opt
sudo wget https://github.com/keycloak/keycloak/releases/download/21.0.0/keycloak-21.0.0.zip
sudo dnf install unzip
sudo unzip keycloak-21.0.0.zip
sudo mv keycloak-21.0.0 keycloak

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
#proxy=edge
#proxy-headers=xforwarded

hostname=your_ip_addr
http-enabled=true
https-enabled=false
http-port=8080
```

Build and verify

```shell
cd /opt/keycloak
sudo -u keycloak ./bin/kc.sh build
sudo -u keycloak KEYCLOAK_ADMIN=admin KEYCLOAK_ADMIN_PASSWORD='test@123' ./bin/kc.sh start --optimized
```

Your Keycloak will start running successfully, navigate to **http://your-vm-ip:8080** and login with your temporary username and password, now you can also start using Keycloak, create users, clients, configure their settings etc. 

Now our next step will be to upgrade our keycloak to the latest stable version, for this few things we need to keep in mind:

1. We can to install a the latest stable version of OpenJDK **(Optional)**
2. Need to create a backup for the SQL database and its configurations.
	+ Refer to https://github.com/CoolSrj06/Fossee-Daily-Progress/blob/master/Keycloak%20BackUp%20%26%20Restore/Keycloak%20Backup.md
3. Download the latest version of Keycloak. 
4. Rename the old Keycloak directory with some other name, so that new one can take its place.

*Lets complete this next part...*