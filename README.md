## Project5-Linux-Server-Configuration ##
Udacity final project 5 Linux server configuration

The object of this project is to secure a linux distribution server and host the web application from the earlier Project 3. On the host provided by Udacity and Amazon I've implemented a list of security features to meet and exceed the required specifications also making sure that the application is fully functional for public use. 
 - to access this instance publically use : http://udacitymarket.no-ip.biz
 - the external IP is : http://52.89.6.106/ but it can only be used to view the website content, authentication will not work as Google's OAuth 2.0 client IDs only accepts fully qualified domain names.

Steps below are explained using unix commands and where necessary commenting reasons behind the choices I have made, including links to some helpful resources. 


##### Step 1 - Add new user grader, setup Key-based authentication, change default SSH port to 2200

 ```
~$ adduser grader
```
add this user to the www-data group so it can install the flask app ( I have restricted access to the www directory to www-data group )
```
~$ usermod -a -G www-data grader
```
add user to sudo:
```
~$ usermod -a -G sudo grader
```
generate ssh keys on another server and copy the public hash into : ~/.ssh/authorized_keys
```
~$ ssh-keygen -t rsa
```
copy them over:
```
~$ scp grader@xxx.xxx.167.124:~/.ssh/gradersrv_rsa.pub .
```
and add:
```
~$cat gradersrv_rsa.pub >> authorized_keys
 ```
 change the ssh port to 2200,first install ssh:
```
~$ sudo apt-get install openssh-server
```
change settings:
```
~$ sudo nano /etc/ssh/sshd_config
```
edit default port 22 to 2200:
```
#What ports, IPs and protocols we listen for
Port 2200
```
also change root login to no:
```
PermitRootLogin no
```
and no passwords:
```
PasswordAuthentication no
```
enable key authentication:
```
PubkeyAuthentication yes
```

 
Resources: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-logwatch-log-analyzer-and-reporter-on-a-vps


##### Step 2 - Configuring local timezone to UTC
Even though the instance from amazon was on UTC already, am listing below the commands for reference:

install with:
```
~$ dpkg-reconfigure tzdata
```
result after :
```
Current default time zone: 'Etc/UTC'
Local time is now:      Fri Oct  2 09:21:31 UTC 2015.
Universal Time is now:  Fri Oct  2 09:21:31 UTC 2015.
```

calling "~$ date" will now show:
```
Fri Oct  2 09:22:28 UTC 2015
```
##### Step 3 - Enable and configure UFW
- Make sure that before UFW is enabled the ssh port 2200 is allowed or risk locking out of your instance!

change ssh port to 2200
```
~$ sudo ufw allow 2200/tcp
```
allow 80, 124
```
~$ sudo ufw allow ntp

~$ sudo ufw allow http
```
enable:
```
~$ sudo ufw enable
```
check status:
```
~$ sudo ufw status
```
```
To                         Action      From
--                         ------      ----
2200/tcp                   ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
123                        ALLOW       Anywhere
2200/tcp (v6)              ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
123 (v6)                   ALLOW       Anywhere (v6)
```
##### Step 4 - Install and configure Apache to serve Python mod_wsgi applications

I have secured the  /var/www/ folder and making it accessible only to users in www-data group, reason why grader is made part of that group earlier.

install the neccessary:
```
~$ sudo aptitude install apache2 apache2.2-common apache2-mpm-prefork apache2-utils libexpat1 ssl-cert
```
install mod-wsgi:
```
~$ sudo aptitude install libapache2-mod-wsgi
```
to enable mod_wsgi, run the following
```
~$ sudo a2enmod wsgi
~$ sudo service apache2 restart
```
create the structure to serve the app, I've used following commands:
```
~$ cd /var/www 
~$ sudo mkdir FlaskApp
~$ sudo mkdir FlaskApp
~$ cd FlaskApp
~$ sudo mkdir static templates
```
My directory structure after this:
```
|----FlaskApp
|---------FlaskApp
|--------------static
|--------------templates
```
In order to make the uploads possible in the static directory I've chmod this to 777

I have also used virtual environment to isolate from the main local host. Installed pip for this:
```
sudo apt-get install python-pip 
sudo pip install virtualenv 
```
Now install Flask and dependencies for the Project 3 app in the virtual environment:
```
source venv/bin/activate 
sudo pip install Flask
pip install Flask-Login
pip install Flask-WTF
pip install flask-seasurf
```

to deactivate the venv:

```
deactivate
```
configuring the Virtual Host :

```
sudo nano /etc/apache2/sites-available/FlaskApp.conf
```

and here is my config, exposing both the IP and registered domain name:

```
<VirtualHost *:80>
                ServerName http://udacitymarket.no-ip.biz
                ServerAdmin admin@52.89.6.106
                WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
                DocumentRoot /var/www/FlaskApp/FlaskApp
                <Directory /var/www/FlaskApp/FlaskApp/>
                        WSGIProcessGroup FlaskApp
                        WSGIApplicationGroup %{GLOBAL}
                        Order allow,deny
                        Allow from all
                </Directory>
                Alias /static /var/www/FlaskApp/FlaskApp/static
                <Directory /var/www/FlaskApp/FlaskApp/static/>
                        Order allow,deny
                        Allow from all
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
                <Directorymatch "^/.*/\.git/">
                        Order deny,allow
                        Deny from all
                </Directorymatch>
</VirtualHost>

<VirtualHost *:80>
                ServerName http://52.89.6.106
                ServerAdmin admin@52.89.6.106
                WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
                DocumentRoot /var/www/FlaskApp/FlaskApp
                <Directorymatch "^/.*/\.git/">
                        Order deny,allow
                        Deny from all
                </Directorymatch>
</VirtualHost>
```
Note that .git is blocked to serve adding this:

```
                <Directorymatch "^/.*/\.git/">
                        Order deny,allow
                        Deny from all
                </Directorymatch>
```


- Important - disable directory browsing! :
```
~$ sudo a2dismod autoindex
```
enable the virtual host:

```
sudo a2ensite FlaskApp
```

creating the .wsgi File :

```
sudo nano flaskapp.wsgi
```

```
#!/usr/bin/python
import sys
import logging
import os

applicationPath = '/var/www/FlaskApp/FlaskApp'
if applicationPath not in sys.path:
    sys.path.insert(0, applicationPath)

os.chdir(applicationPath)

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/FlaskApp/FlaskApp/")

from project import app as application
application.secret_key = 'xxxxxxxxxxxxxxxxxxx'

```

I've installed Git so I can clone the repository, I have cloned it in my home directory and copy the files
across to /www/FlaskApp/FlaskApp/ directory manually.

restart apache:
```
sudo service apache2 restart 
```

Resources:
http://fideloper.com/user-group-permissions-chmod-apache
https://www.digitalocean.com/community/tutorials/installing-mod_wsgi-on-ubuntu-12-04

##### Step 4 - Install PosgreSQL and secure it

install postgres:
```
sudo apt-get install postgresql
```
also to install psycopg2, we need libpq-dev:
```
sudo apt-get install libpq-dev
pip install psycopg2
```
creating the catalog user, the default superuser for PostgreSQL is called postgres we will use it to create the new role:

```
$ sudo su - postgres
postgres$ createuser --createdb --username postgres --no-createrole --pwprompt catalog
Enter password for new role: 
Enter it again: 
Shall the new role be a superuser? (y/n) y
CREATE ROLE
postgre
```
securing postgres, no remote. For this we are looking into the host based authentication file:

```
sudo nano /etc/postgresql/9.1/main/pg_hba.conf
```
```
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            md5
#host    replication     postgres        ::1/128                 md5
```
here we see that only local IPs are allowed.

Reference:
https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
