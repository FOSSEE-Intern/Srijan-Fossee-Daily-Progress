# Task

Date: 11/10/25

- 18:30 Logged in
- 18:40 Download VirtualBox and RockyLinux 10 image
- 19:20 Successfully installed rockylinux machine on VB
- 19:30 Updated the packages of rockylinux
- 19:40 Install required dependencies
- 20:00 Setup project dir with files init_sites, sites.json, Dockerfile and openplc.sql
- 20:10 Ran init_sites
- 20:20 The container is up and running and the site is accessible on hiting the port at 8080.
- 20:50 Installed kecloak
- 23:30 Configure keycloak and drupal with SSO
- 00:00 Stuck into error on Setting up SSO.
- 04:00 Fixed the issue with correct IP, switched from using localhost to using public IPv4, assigned to my VM
- 4:10 Accessed from the host via ssh (Need to install the required dependencies from ssh).

**Next Day**
+ 12:30 Stopped and rebuid the openplc websites.
+ 12:45 Stopped the keycloak, reconfigured the keycloak.conf file by changing the hostname value to public ip of vm.
+ 12:50 Rebuild the keycloak and Ran it on the VM (Systemctl).
+ 13:00 Now configured SSO the website. 
+ 13:10 SSO setup successful.
+ 14:00 Documented the Task.md
+ 15:00 Finalize task documentation
+ 15:30 Logged off