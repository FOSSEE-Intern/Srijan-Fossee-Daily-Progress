## Day1 Logs

**Running openplc on Rocky Linux**
---
+ 16:00: Created an instance for Rocky Linux 8.
+ Logged in.
+ 16:05: Installed the neccesary packages (podman, slirp4netns, jq, mysql) and setup dir for openplc
+ 16:10: Created sites.json file and added the required json content.
+ 16:15: Executed the ./init_scripts, the script ended with and error
+ 16:17: Rectified that error and finally the container up and running.
+ 17:00: Acceessed the container at port 8081.
+ 17:05: The website return and internal server error.
+ 17:10: On checking logs of the container found that the there was some connection database issue.
+ 20:00: Finally after a lots of tries and reseach, reached at the conclusion that the issue was being caused by podman. 
+ Finally, was recommended to deploy the site on Ubuntu.
