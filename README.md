## Project5-Linux-Server-Configuration ##

Udacity final Project 5 Linux server configuration

###### The object of this project is to secure a Linux distribution server and host the web application from the earlier Project3. On the host provided by Udacity and Amazon I've implemented a list of security features to meet and exceed the required specifications also making sure that the application is fully functional for public use. I have also installed and scheduled cron jobs with applications that automatically download updates and upgrade the system unattended.

 - for public access to this instance use: http://udacitymarket.no-ip.biz
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
Note that .git is prevented to be accidently served by adding this:

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
securing postgres - no remote. For this we are looking into the host based authentication file:

```
sudo nano /etc/postgresql/9.3/main/pg_hba.conf
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

Important postgres security issue, checking to make sure db is listening to localhost only, NOT ‘*’ :
```
/etc/postgresql/9.3/main/postgresql.conf
```
```
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

#listen_addresses = 'localhost'
```
Reference:
https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps

#### Extra work done to harden the security of the server, update and upgrade packages automatically, also email alerting providing server's vital status or any security alerts.

##### Automatic Package Updates

* cron-apt method implementation

cron-apt is designed to automatically update the package list and download upgraded packages.

```
sudo apt-get install cron-apt
```
this will be croned every day at 4AM:

```
0 4  * * * root test -x /usr/sbin/cron-apt && /usr-sbin/cron-apt
```
Reference:
https://help.ubuntu.com/community/AutoWeeklyUpdateHowTo


 * unattended-upgrades package method implementation

installed with:

```
sudo apt-get install unattended-upgrades
```
configured with :

```
sudo dpkg-reconfigure --priority=low unattended-upgrades
```
which will create /etc/apt/apt.conf.d/20auto-upgrades with the following contents:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```
I have enabled sending emails only if there are any errors modifying:
```
/etc/apt/apt.conf.d/50unattended-upgrades
```
changing it to my email account:
```
Unattended-Upgrade::Mail "george@mydomain.com”
Unattended-Upgrade::MailOnlyOnError "true";
```
but no Reboot
```
Unattended-Upgrade::Automatic-Reboot "false";
```
To test I am running this :
```
sudo unattended-upgrade -v -d --dry-run
```
Note: You may need to install sendmail to be able to send emails from the server :
```
sudo apt-get install mailutils
sudo apt-get install sendmail
```
After the automatic updates/upgrades I would get an email like this:
```
“Unattended upgrade returned: True

Warning: A reboot is required to complete this upgrade.

Packages that were upgraded:
 cloud-init grub-legacy-ec2 libcgmanager0 python3-software-properties
 software-properties-common
. . . 
```

Reference:
https://help.ubuntu.com/community/AutomaticSecurityUpdates

#####  Monitoring

* Logwatch installed to monitor all activities from the server and send emails with reports regulary

installed with:
```
aptitude install -y logwatch
```
configuring with:
```
nano /usr/share/logwatch/default.conf/logwatch.conf
```
changes that I've made:
```
MailTo = myemail@dot.com
Range = today
Detail = Medium
```
example of reports I receive:
```
################### Logwatch 7.4.0 (05/29/13) ####################
        Processing Initiated: Thu Oct  1 15:12:04 2015
        Date Range Processed: today
                              ( 2015-Oct-01 )
                              Period is day.
        Detail Level of Output: 0
        Type of Output/Format: mail / text
        Logfiles for Host: ip-10-20-26-195
 ##################################################################

 --------------------- httpd Begin ------------------------

 Requests with error response codes
    400 Bad Request
       /x: 1 Time(s)
    403 Forbidden
       /admin: 2 Time(s)
       /: 1 Time(s)
    404 Not Found
       /cgi-bin/php: 2 Time(s)
       /cgi-bin/php-cgi: 2 Time(s)
       /cgi-bin/php.cgi: 2 Time(s)
       /cgi-bin/php4: 2 Time(s)
       /cgi-bin/php5: 2 Time(s)
       /favicon.ico: 1 Time(s)
       /manager/html: 1 Time(s)
       /xmlrpc.php: 1 Time(s)
    500 Internal Server Error
       /forsale/1/new: 4 Time(s)
       /admin/: 2 Time(s)
       /favicon.ico: 2 Time(s)
       /: 1 Time(s)

 ---------------------- httpd End -------------------------


 ###################### Logwatch End #########################

```
Reference:
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-logwatch-log-analyzer-and-reporter-on-a-vps

#####  Security hardening - Monitoring firewall attacks

 - I’ve installed and enabled ModSecurity Web Application Firewall (WAF):
```
apt-get install libapache2-modsecurity
```
verify if it works:
```
apachectl -M | grep --color security
```
Configuring, rename the default :
```
mv /etc/modsecurity/modsecurity.conf{-recommended,}
```
edit modsecurity.conf: nano /etc/modsecurity/modsecurity.conf

```
SecRuleEngine On
```
Not to use resources turn off this :
```
SecResponseBodyAccess Off
```
We could limit the data uploads but because I have an upload request on my site I left these as default:
```
SecRequestBodyLimit
SecRequestBodyNoFilesLimit
```
Load the default rules and tell Apache to look at this locations :
```
sudo nano /etc/apache2/mods-enabled/modsecurity.conf
```
```
Include "/usr/share/modsecurity-crs/*.conf"
Include "/usr/share/modsecurity-crs/activated_rules/*.conf"
```
Symlinks must be created inside the activated_rules directory to activate these. Let us activate the SQL injection rules.
```
cd /usr/share/modsecurity-crs/activated_rules/
ln -s /usr/share/modsecurity-crs/base_rules/modsecurity_crs_41_sql_injection_attacks.conf .
```
Apache has to be reloaded for the rules to take effect.
```
service apache2 reload
```
Important Note:

My /etc/apache2/mods-enabled/modsecurity.conf example :
```
Include "/usr/share/modsecurity-crs/*.conf"
Include "/usr/share/modsecurity-crs/activated_rules/*.conf"

<Directory "/var/www/FlaskApp/">
    <IfModule security2_module>
        SecRuleEngine Off
    </IfModule>
</Directory>
```
Am excluding FlaskApp otherwise will not be accessible for queries.

 - I’ve also added ModEvasive preventing DDOs attacks :

install and configure permissions also create a log file directory:
```
sudo apt-get install libapache2-mod-evasive
sudo mkdir /var/log/mod_evasive
sudo chown www-data:www-data /var/log/mod_evasive/
```
add the following to : /etc/apache2/mods-available/mod-evasive.conf
```
<ifmodule mod_evasive20.c>
   DOSHashTableSize 3097
   DOSPageCount  2
   DOSSiteCount  50
   DOSPageInterval 1
   DOSSiteInterval  1
   DOSBlockingPeriod  10
   DOSLogDir   /var/log/mod_evasive
   DOSEmailNotify  EMAIL@DOMAIN.com
   DOSWhitelist   127.0.0.1
</ifmodule>
```


check if it’s enabled and restart apache :
```
sudo a2enmod evasive
sudo service apache2 restart
```
Resources:
https://www.digitalocean.com/community/tutorials/how-to-set-up-mod_security-with-apache-on-debian-ubuntu
https://www.thefanclub.co.za/how-to/how-install-apache2-modsecurity-and-modevasive-ubuntu-1204-lts-server

#####  Security hardening - Monitoring brute force login attacks

 - I have also installed Fail2ban on my instance. There is a strong possibility to lock yourself out so by adding my ip to  /etc/fail2ban/jail.local had minimized that.

install :
```
sudo apt-get update
sudo apt-get install fail2ban
```
copy and changed the local configuration:
```
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
changed the ssh entry on port 2200 to ban any failed attempts:
``` 
[ssh]

enabled  = true
port     = 2200
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
```

to receive email alerts every time there is a potential attack, I've changed from :
```
action = %(action_)s
```
to
```
action = %(action_mw)s
```

Added my IP to ignore list , just in case am locking myself out:
```
ignoreip = 127.0.0.1/8  <my ip>
```
set my email in send mail:
```
destemail = root@localhost
```
to
```
destemail = myemail@dot.com
```
Tested by trying to log with an unregistered user and got banned, checking with :
```
sudo iptables -S
```
you will see an entry like this :
```
-A fail2ban-ssh -s xx.xxx.12.198/32 -j REJECT --reject-with icmp-port-unreachable
```
which means that my attempt was recorded and my IP is now banned, also receiving the following email :
```
Hi,
The IP xx.xxx.12.198 has just been banned by Fail2Ban after
3 attempts against ssh.
```
Resource:
https://www.digitalocean.com/community/tutorials/how-to-protect-an-apache-server-with-fail2ban-on-ubuntu-14-04

