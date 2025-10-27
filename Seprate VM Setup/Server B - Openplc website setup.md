## Setup guide for Drupal Site

## 1. Install required dependencies

Update and configure firewall

```sh
sudo dnf update -y

# allow port 8081
sudo firewall-cmd --permanent --add-port=8081/tcp

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

Install podman, jq and slirp4netns

```sh
sudo dnf install wget git podman jq slirp4netns -y
```

## 2. Setup project directory

Create a directory

```sh
mkdir openplc

git clone https://github.com/CoolSrj06/Fossee-Daily-Progress.git
cp Fossee-Daily-Progress/Openplc/Dockerfile openplc/
cp Fossee-Daily-Progress/Openplc/init.sites openplc/

cd openplc
touch sites.json
```

Add the below content in sites.json
```bash
[
  {
    "SITE_NAME": "openplc",
    "PORT": 8081,
    "REPO": "https://github.com/FOSSEE/openplc_docker_image"
  }
]
```
Run init_sites script

You will also need openplc.sql, a sql file that will be have been shared to you. If you need to tranfer it from your host machine(Windows) to your vm you can use this format of command

```
scp openplc.sql username@<public-ip>:~/
```

If any error fix it, might get scp not found error, if yes, fix it.

```sh
# if file is not executible the run the below command or else ignore it.
chmod +x init_sites
./init_sites
```

Enter the fields

```sh
Enter MySQL root password: {your_sql_root_password}
!! Kindly use lowercase for naming and don't include white spaces !!
Enter name of the site: openplc
Enter name of site database: openplcdb
Add date tag to database name? [default: _2025_10_10]:
Enter database username you wish to use: openplcuser
Enter path to SQL dumpfile for site: ./openplc.sql
Choose Drupal version for site (10, 11): 10
Enter branch name to clone: main

Is this information correct? (y/n): y

Do you want to create and use a new volume? [y/N]: y

Enter custom volume name [default: openplc_volume]:

Proceed with image build? (y/N): y

```

Check if container is running

```sh
podman ps
```

Create openplc user

```bash
podman exec -it openplc_container bash
chmod +x vendor/bin/drush
vendor/bin/drush cr # Reload cache
vendor/bin/drush user-create {your_username} --mail="{your_email}" --password="{your_password}"
vendor/bin/drush user-add-role "administrator" {your_username}
exit
```
## 3. Configure Drupal 

Drop into container and install open_id and keycloak modules

```bash
podman exec -it openplc_container bash
composer require drupal/openid_connect drupal/keycloak
# Enable modules using drush
vendor/bin/drush en openid_connect keycloak -y
vendor/bin/drush cr
exit
```

### Drupal: Get Redirect URL

- Open your Drupal site.
- Login and navigate to Configuration > Web Services > OpenID Connect.
- Under Enabled OpenID Connect clients check the Keycloak
- Copy Redirect url given

### Final Configuration

- At Drupal's Configuration > OpenID Connect.
  - Client ID: `openplc`
  - Client secret: `{client_secret}` // paste the secret copied from Keycloak.
  - Keycloak base url: `http://{keycloak_domain}:8080`
  - Keycloak realm: `master` (or your custom realm).
- Enable these options:

  - Replace Drupal login with Keycloak single sign-on (SSO)
  - Enable Drupal-initiated single sign-out
  - Enable Keycloak-initiated single sign-out

- Save configuration
- Navigate to User > View profile > Connected Accounts > Connect Keycloak to link your existing Drupal admin user to Keycloak.
- Logout from the Drupal site.

## 4. Test SSO

Rebuild drupal site cache

```sh
# Drop into container
podman exec -it openplc_container bash
# Refresh rebuild cache
vendor/bin/drush cr
exit
```

Login using SSO

- Logout from keycloak console
- Now open http://{openplc_domain}/user/login
- You should be redirected to keycloak console
- Login with your keycloak credentials
- On successful login you will be redirected back to Openplc site

### 4. Creating Podman Image

We will create an new image which will be have the modified configurations and updates.

+ First of all stop the running container.
+ Now re-run the image with 

```bash
podman run -d --name openplc_container --cgroups=split --pull never --network slirp4netns:allow_host_loopback=true --cap-add net_raw -v openplc_volume:/var/www/html/sites/default/files:Z --publish 8081:80   localhost/openplc_image:latest
```

Once the container is up, we need to perform some operations inside the container that we did earlier too.

```bash
podman exec -it openplc_container bash
composer require drupal/openid_connect drupal/keycloak
# Enable modules using drush
vendor/bin/drush en openid_connect keycloak -y
vendor/bin/drush cr
exit
```

Once that's done, create a new image, I will use the same name as the current one, as of which next time the openplc_persist.service file is run the image will run and it will be the updated image. 

```bash
podman commit openplc_container localhost/openplc_image
podman images
```

Now run openplc_persist.service
```bash
systemctl --user start openplc_persist
```

Now the container will start running, also now anytime if the server restarts the container will run according to the updated state, and errors that we recieved earlier will not follow.