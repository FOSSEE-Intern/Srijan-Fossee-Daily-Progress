## Running Openplc in Ubuntu Instance

### Prerequisites
+ Login to your instance.
+ Install **podman, jq and slirp4netns**
+ Install MySQL: Documentation Refercence
 https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-ubuntu-20-04 


### Step 1
+ Create a json file sites.jon and insert
```bash
[
  {
    "SITE_NAME": "openplc",
    "PORT": 8081,
    "REPO": "https://github.com/FOSSEE/openplc_docker_image"
  }
]
```
+ Import **Dockerfile** and **init_sites**

+ Now move these three files into newly created directry(let name: *Main*)
+ Also add openplc.sql file in this dir provided by the instructor.

### Step 2
+ Execute 

```bash
./init_scripts
```

+ **Enter the site configurations**
```bash
Enter MySQL root password: {your_sql_root_password}
!! Kindly use lowercase for naming and don't include white spaces !!
Enter name of the site: openplc
Enter name of site database: openplc_db_srijan
Add date tag to database name? [default: _2025_10_10]:  
Enter database username you wish to use: openplc_srijan
Enter path to SQL dumpfile for site: ./openplc.sql
Choose Drupal version for site (10, 11): 10
Enter branch name to clone: main

Summary of entered information:
Site Name        : openplc
Database Name    : openplcdb_2025_10_10
Database Username: openplc_srijan
SQL Dump File    : ./openplc.sql
Image Name       : openplc_image
Container Name   : openplc_container
Service Name     : openplc_persist
Drupal Version   : 10

Is this information correct? (y/n): y
Confirmed. Continuing...
[sudo] password for root: 
```

The script will run and you should get no error.

### Step 3
+ Visit http://[instance-ip]:8081
+ Website should be accessible

### Step 4

+ **Enter in Podman Container**

```bash
podman exec -it openplc_container bash
chmod +x vendor/bin/drush
vendor/bin/drush cr # Reload cache
vendor/bin/drush user-create {your_username} --mail="{your_email}" --password="{your_password}"
vendor/bin/drush user-add-role "administrator" {your_username}
```

+ Finally revisit the site and login with the user credentials.

----------
