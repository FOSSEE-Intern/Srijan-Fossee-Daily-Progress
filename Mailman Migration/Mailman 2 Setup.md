## 1. VM Setup and Network Configuration

### Download iso files

- Centos 7: https://vault.centos.org/7.9.2009/isos/x86_64/CentOS-7-x86_64-Minimal-2207-02.iso

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

## 2. CentOS 7 Setup

### Add your user to sudoers

```sh
su -
usermod -aG wheel your_centos_user
exit
exit
```

Log in again with your user account and verify sudo access:

```sh
sudo yum update -y
```

### Configure CentOS Vault Repository

Backup existing repository settings:

```sh
mkdir -p /etc/yum.repos.d/old
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/old/
```

Create CentOS Vault repository configuration:

```sh
sudo vi /etc/yum.repos.d/CentOS-Vault.repo
```

Add the following content:

```ini
[base]
name=CentOS-7.9.2009 - Base
baseurl=http://vault.centos.org/7.9.2009/os/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[updates]
name=CentOS-7.9.2009 - Updates
baseurl=http://vault.centos.org/7.9.2009/updates/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[extras]
name=CentOS-7.9.2009 - Extras
baseurl=http://vault.centos.org/7.9.2009/extras/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[centosplus]
name=CentOS-7.9.2009 - CentosPlus
baseurl=http://vault.centos.org/7.9.2009/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
```

Create EPEL repository configuration:

```sh
sudo vi /etc/yum.repos.d/epel.repo
```

Add the following content:

```ini
[epel]
name=Extra Packages for Enterprise Linux 7
baseurl=https://archives.fedoraproject.org/pub/archive/epel/7/$basearch
enabled=1
gpgcheck=1
gpgkey=https://archives.fedoraproject.org/pub/archive/epel/RPM-GPG-KEY-EPEL-7
```

Clean cache and update:

```sh
sudo yum clean all
sudo yum makecache
sudo yum update -y
```

## 3. Install Postfix and Dovecot

### Install and Configure Postfix

```sh
sudo yum install -y postfix
sudo vi /etc/postfix/main.cf
```

Edit the following lines:

```ini
# Uncomment and specify hostname
myhostname = mail.srv.world

# Uncomment and specify domain name
mydomain = srv.world

# Uncomment
myorigin = $mydomain

# Change
inet_interfaces = all

# Add
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

# Uncomment and specify your local network
mynetworks = 127.0.0.0/8, 10.0.0.0/24

# Uncomment (use Maildir)
home_mailbox = Maildir/

# Add
smtpd_banner = $myhostname ESMTP

# Add at the end
message_size_limit = 10485760
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $myhostname
smtpd_recipient_restrictions = permit_mynetworks, permit_auth_destination, permit_sasl_authenticated, reject
```

### Install and Configure Dovecot

```sh
sudo yum install dovecot -y
sudo vi /etc/dovecot/dovecot.conf
```

Edit the following lines:

```ini
# Uncomment
protocols = imap pop3 lmtp

# Uncomment and change
listen = *
```

Edit `/etc/dovecot/conf.d/10-auth.conf`:

```sh
sudo vi /etc/dovecot/conf.d/10-auth.conf
```

```ini
# Uncomment and change
disable_plaintext_auth = no

# Add
auth_mechanisms = plain login
```

Edit `/etc/dovecot/conf.d/10-mail.conf`:

```sh
sudo vi /etc/dovecot/conf.d/10-mail.conf
```

```ini
# Uncomment and add
mail_location = maildir:~/Maildir
```

Edit `/etc/dovecot/conf.d/10-master.conf`:

```sh
sudo vi /etc/dovecot/conf.d/10-master.conf
```

```ini
# Uncomment and add
unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
}
```

Edit `/etc/dovecot/conf.d/10-ssl.conf`:

```sh
sudo vi /etc/dovecot/conf.d/10-ssl.conf
```

```ini
# Change
ssl = no
```

Start and enable services:

```sh
sudo systemctl start dovecot
sudo systemctl enable dovecot
sudo systemctl restart postfix
sudo systemctl enable postfix
```

Configure firewall:

```sh
sudo firewall-cmd --add-service=smtp --permanent
sudo firewall-cmd --add-service={pop3,imap} --permanent
sudo firewall-cmd --reload
```

## 4. Install Apache Web Server

```sh
sudo yum install httpd -y
sudo rm -f /etc/httpd/conf.d/welcome.conf
```

Edit Apache configuration:

```sh
sudo vi /etc/httpd/conf/httpd.conf
```

Update the following lines:

```apache
# Change to admin's email address
ServerAdmin root@srv.world

# Change to your server's name
ServerName www.srv.world:80

# Change
AllowOverride All

# Add file name that it can access only with directory's name
DirectoryIndex index.html index.cgi index.php

# Add at the end
ServerTokens Prod
```

Start and enable Apache:

```sh
sudo systemctl start httpd
sudo systemctl enable httpd
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
```

## 5. Install and Configure Mailman 2

### Install Mailman

```sh
sudo yum install mailman -y
```

### Configure Mailman

Edit main configuration:

```sh
sudo vi /etc/mailman/mm_cfg.py
```

```python
# Change to your hostname with FQDN
DEFAULT_URL_HOST = 'mail.srv.world'
DEFAULT_EMAIL_HOST = 'mail.srv.world'
```

Edit defaults configuration:

```sh
sudo vi /usr/lib/mailman/Mailman/Defaults.py
```

Update the following settings:

```python
# Change default MTA
MTA = 'Postfix'

# Change to your language
DEFAULT_SERVER_LANGUAGE = 'en'

# Change if needed
OWNERS_CAN_DELETE_THEIR_OWN_LISTS = Yes

# Set action for non-member posts
# 0 = Accept, 1 = Hold, 2 = Reject, 3 = Discard
DEFAULT_GENERIC_NONMEMBER_ACTION = 1

# Reply-To settings
# 0 - Not munged, 1 - Set to list, 2 - Set to explicit value
DEFAULT_REPLY_GOES_TO_LIST = 0
```

### Initialize Mailman

Generate alias file:

```sh
sudo /usr/lib/mailman/bin/genaliases
```

Set Mailman admin password:

```sh
sudo /usr/lib/mailman/bin/mmsitepass
```

Check and fix permissions:

```sh
sudo /usr/lib/mailman/bin/check_perms
sudo /usr/lib/mailman/bin/check_perms -f
sudo /usr/lib/mailman/bin/check_perms
```

Fix alias permissions:

```sh
sudo chown apache /etc/mailman/aliases
sudo chmod 664 /etc/mailman/aliases*
```

Create administrative mailing list:

```sh
sudo /usr/lib/mailman/bin/newlist mailman
```

Follow the prompts to enter email and password.

Start and enable Mailman:

```sh
sudo systemctl start mailman
sudo systemctl enable mailman
```

## 6. Export Data from CentOS 7

### Create Migration Directory

```sh
mkdir -p /tmp/mm2_migration
```

### Export List Configurations

```sh
# Copy list config files (pickle format)
sudo cp -r /var/lib/mailman/lists/ /tmp/mm2_migration/
```

### Export Archives

```sh
# Copy mbox archive files
sudo cp -r /var/lib/mailman/archives/private/ /tmp/mm2_migration/
```

### Create Compressed Archive

```sh
# Change to /tmp directory
cd /tmp

# Create compressed tar archive
sudo tar -czvf mailman2_migration_data.tar.gz mm2_migration/
```

### Transfer to AlmaLinux

```sh
# Transfer compressed archive to AlmaLinux server
scp /tmp/mailman2_migration_data.tar.gz user@192.168.56.11:/tmp/
```

Replace user and IP address with your AlmaLinux server details.

*This `tar` contains the configuration and archives or mail man 2.*