# Project5-Linux-Server-Configuration
Udacity final project 5 Linux server configuration

##### The object of this project is to secure a linux distribution server and host the web application from the earlier Project 3. On the host provided by Udacity and Amazon I've implemented a list of security features to meet and exceed the required specifications also making sure that the application is fully functional for public use. 
 - to access this instance publically use : http://udacitymarket.no-ip.biz
 - the external IP is : http://52.89.6.106/ but it can only be used to view the website content as Google's OAuth 2.0 client IDs only accepts fully qualified domain names.

##### Steps below are explained using unix commands and where neccessary commenting reasons behind the choices I have taken

###### Step 1 - Add new user grader, setup Key-based authentication, change default SSH port to 2200

#adduser grader
add this user to the www-data group so it can install the flask app ( I have restricted access to the www directory to www-data group )
#usermod -a -G www-data grader
add user to sudo:
#usermod -a -G sudo grader
generate ssh keys on another server and copy the public hash into
~/.ssh/authorized_keys
#ssh-keygen -t rsa
copy them over:
#scp grader@xxx.xxx.167.124:~/.ssh/gradersrv_rsa.pub .
and add:
#cat gradersrv_rsa.pub >> authorized_keys


