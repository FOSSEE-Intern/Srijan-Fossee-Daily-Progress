# 1. What are we trying to do ?

+ **Take backup** of your **Keycloak server** and its **database** (daily and manually).
+ **Store** those backups somewhere safe (like `/backups` folder or a remote VM/cloud).
+ **If there is a disaster** (server crash / VM loss).
+ **Restore** Keycloak on a new VM using the backup you made.

So, when you restore successfully — it should look like:
- All users, realms, and configurations are the same as before.

## What to Backup in Keycloak

+ Configuration files & deployment (Application layer)
+ Database

# 2. Create a Backup Script

## 2.1 Make a Backup Folder

```bash
sudo mkdir -p /var/backups/keycloak
sudo chown $(whoami):$(whoami) /var/backups/keycloak
```

## 2.2 Backup Script

Create a file called **`/usr/local/bin/keycloak-backup.sh`:**

```bash
#!/bin/bash

BACKUP_DIR="/var/backups/keycloak"
DATE=$(date +"%Y-%m-%d_%H-%M-%S")
DB_NAME="Keycloak"
DB_USER="keycloak"
DB_HOST="localhost"
DB_PASSWORD="keycloak-password"
KEYCLOAK_DIR="/opt/keycloak"

mkdir -p "${BACKUP_DIR}/${DATE}"

echo "Backing up Keycloak database..."
mysqldump -u "${DB_USER}" -p"${DB_PASSWORD}" -h "${DB_HOST}" "${DB_NAME}" > "${BACKUP_DIR}/${DATE}/keycloak-db.sql"

if [ $? -ne 0 ]; then
  echo "ERROR: mysqldump failed"
  exit 1
fi


echo "Backing up Keycloak configuration..."
tar -czf "${BACKUP_DIR}/${DATE}/keycloak-config.tar.gz" "${KEYCLOAK_DIR}/conf"

if [ $? -ne 0 ]; then
  echo "ERROR: tar failed"
  exit 1
fi

echo "Backup completed: ${BACKUP_DIR}/${DATE}"
```

Give permission to run:

```bash
sudo chmod +x /usr/local/bin/keycloak-backup.sh
```

### 2.2.1

| Variable      | Meaning                                                                       |
| ------------- | ----------------------------------------------------------------------------- |
| `DB_NAME`     | The **name of the database** used by Keycloak                                 |
| `DB_USER`     | The **username** that connects to that database                               |
| `DB_HOST`     | The **host address** of the database (usually `localhost` if on same machine) |
| `DB_PASSWORD` | The keycloak database password                                                |
DB_USER, DB_HOST, DB_PASSWORD can be obtained from **cat /opt/keycloak/conf/keycloak.conf**, 
where as for DB_NAME

```bash
sudo mysql -u root -p
SHOW DATABASES;
```

You might see:

```bash
+--------------------+
| Database           |
+--------------------+
| keycloak           |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+

```

Therefore **DB_NAME = keycloak**

## 2.3 Test is Manually

```bash
/usr/local/bin/keycloak-backup.sh
```
This will create a folder like:
```swift
`/var/backups/keycloak/2025-11-08_18-00-00/  
├── keycloak-db.sql  
└── keycloak-config.tar.gz`
```

# 2.4 Automate Daily Backup via Cron

Open cron:
```bash
crontab -e
```

Add this line:
```bash
0 2 * * * /usr/local/bin/keycloak-backup.sh >> /var/log/keycloak-backup.log 2>&1
```

This will run everyday at 2 AM