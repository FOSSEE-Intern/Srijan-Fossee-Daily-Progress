Reference: https://www.keycloak.org/docs/latest/upgrading/index.html

1. Stop the running Keycloak.
2. Create a backup for sql database and keycloak configuration. 
	+ Refer to https://github.com/CoolSrj06/Fossee-Daily-Progress/blob/master/Keycloak%20BackUp%20%26%20Restore/Keycloak%20Backup.md
3. Download New Stable version of Keycloak. 
4. Configure it.
5. Build and Run 

## Prerequisites

```bash 
cd /opt/
sudo mv keycloak keycloak_old
```

## Download and Install Keycloak

```bash
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

Download MySQL JDBC Driver and add it into keycloak

```shell
cd /opt/keycloak/providers
sudo wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.4.0/mysql-connector-j-8.4.0.jar
sudo chown keycloak:keycloak mysql-connector-j-8.4.0.jar
# SELinux file context for provider JAR
sudo chcon -u system_u -t usr_t /opt/keycloak/providers/mysql-connector-j-8.4.0.jar
```

Copy the keycloak.conf file from /opt/keycloak/conf/keycloak.conf or extract and add it from the newly created backup.

Next, create a temporary admin and run keycloak
```bash
cd /opt/keycloak
sudo -u keycloak ./bin/kc.sh build

sudo -u keycloak KEYCLOAK_ADMIN=admin KEYCLOAK_ADMIN_PASSWORD='test@123' ./bin/kc.sh start 
```

Your new version of keycloak should run and work perfectly fine.

*Note: I haven't upgraded my OpenJDK-17 to OpenJDK-21. This will not cause any issue in upgrading keycloak, until the specific version of keycloak supports it.*