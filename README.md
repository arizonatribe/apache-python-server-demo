# Udacity Linux Server Configuration Project
## Overview
The project demonstrates the setup of an AWS virtual machine as a web server application.
Overall it required setting up a virtual machine with many security constraints and logging
capabilities in addition to configuring it as an apache web server to run a Python web
application written in the Flask framework (that was formerly created under a local Vagrant
virtual machine for a previous project).

## Official requirements
1. Launch your Virtual Machine with your Udacity account and log in. 
2. Create a new user named grader and grant this user sudo permissions.
3. Update all currently installed packages.
4. Configure the local timezone to UTC.
5. Change the SSH port from 22 to 2200
6. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
7. Install and configure Apache to serve a Python mod_wsgi application
8. Install and configure PostgreSQL:
    * Do not allow remote connections
    * Create a new user named catalog that has limited permissions to your catalog application database
9. Install git, clone and set up your Catalog App (restaurant menu) project (from your GitHub repository from earlier in
the Nanodegree program) so that it functions correctly when visiting your server’s IP address in a browser)
10. Access your finished web application at your Amazon EC2 Instance's public URL: http://ec2-52-34-56-206.us-west-2.compute.amazonaws.com/login


## Project dependencies
* AWS VM instance

### AWS VM Instance
You can manage your virtual server at: https://www.udacity.com/account#!/development_environment 

# Summary of the Steps involved in completion of this project

## CREATE USER 
create the grader user and group
```bash
useradd grader
```
grant the grader user sudo ability
```bash
usermod -a -G sudo grader
```
Set their password to a secure value 
```bash
passwd grader <secure password>
```

## SSH CONFIGURATION
create the /home/grader/.ssh directory
```bash
mkdir /home/grader/.ssh
```
set the correct permssions for that new folder
```bash
chmod 755 /home/grader/.ssh
```
place the appropriate public key into an authorized_key file
```bash
mv ~/.ssh/authorized_keys /home/grader/.ssh/authorized_keys
```
make sure that the group and user owners are changed for that file
```bash
chown grader:grader /home/grader/.ssh/authorized_keys
```
restarted ssh service
```bash
service ssh restart
```
attempted login with new user
```bash
ssh -i ~/.ssh/udacity_key.rsa grader@52.34.56.206
```
after verifying successfully connected, verified ability to sudo
```bash
sudo -v
```
although password authentication worked for sudo, got an 
"unable to resolve host ip-10-20-44-7" error, so added this line to /etc/hosts
```
127.0.0.1 ip-10-20-44-7
```

modify the `/etc/ssh/sshd_config` file, changing "Port 22" to "Port 2200"
Also, double-checked that:
```
PubkeyAuthentication yes
PasswordAuthentication no
```

```bash
vim /etc/ssh/sshd_config
```
verified ssh worked with new port
```bash
ssh -i ~/.ssh/udacity_key.rsa grader@52.34.56.206 -p 2200
```

## FIREWALL
checked that ufw firewall status inactive
```bash
sudo ufw status
```
set defaults to deny incoming and allow outgoing traffic
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
Allow ssh connections for custom port 2200
```bash
sudo ufw allow 2200/tcp
```
Allow ntp connections
```bash
sudo ufw allow 123/udp
```
Allow standard web access on port 80
```bash
sudo ufw allow www
```
Turn on ufw firewall, bypassing prompt
```bash
sudo ufw --force enable
```

wasn't sure if I needed to reboot, based on message at last prompt, so I did
```bash
reboot
```
reattempted connection successfully
```bash
ssh -i ~/.ssh/udacity_key.rsa grader@52.34.56.206 -p 2200
```

## SECURING SSH, APACHE WITH FAIL2BAN

install logging tool for firewall
```bash
sudo apt-get install fail2ban
```
created a copy of the jail.conf for fail2ban, called jail.local
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
checked that the ssh banning rules are enabled, and then enabled several
apache rules, including: apache, apache-noscript, apache-overflows
also created rules for: apache-badbots, apache-nohome
```bash
sudo vim /etc/fail2ban/jail.local
```

restarted fail2ban service
```bash
sudo service fail2ban restart
```

## TIME CONFIGURATION

Set timezone, selecting "None" followed by "UTC"
```bash
sudo dpkg-reconfigure tzdata
```
setup initial synchronization to time-sync server, to make sure we're close
```bash
sudo ntpdate pool.ntp.org
```
install the ntp daemon
```bash
sudo apt-get install ntp
```
made sure ntp is up and running
```bash
sudo service ntp status
```
check for the list of ntp servers to use: http://www.pool.ntp.org/zone/us
and after commenting out the default ones, placed them in there instead
```bash
vim /etc/ntp.conf
```
restart the service to apply the changes
```bash
sudo service ntp status
```
after waiting a few minutes, checked to see that the servers we're using
look like they might be U.S. or just North America based ("st" column shows a
low value)
```bash
ntpq -p
```

## PACKAGE MANAGEMENT

refresh the list of available packages
```bash
sudo apt-get update
```
upgrade packages to their latest
```bash
sudo apt-get upgrade -y
```
remove unnecessary packages
```bash
sudo apt-get autoremove
```

created a weekly cron script to update and upgrade packages, pasting the
following code into the script file
```bash
sudo vim /etc/cron.weekly/apt-security-updates
```

```bash
echo "**************" >> /var/log/apt-security-updates
date >> /var/log/apt-security-updates
aptitude update >> /var/log/apt-security-updates
aptitude safe-upgrade -o Aptitude::Delete-Unused=false --assume-yes --target-release `lsb_release -cs`-security >> /var/log/apt-security-updates
echo "Security updates (if any) installed"
```

made the script executable
```bash
sudo chmod +x /etc/cron.weekly/apt-security-updates
```
pasted the following code into a logrotate script to control the growth of 
the update log file
```bash
sudo vim /etc/logrotate.d/apt-security-updates
```

```bash
/var/log/apt-security-updates {
        rotate 3
        weekly
        size 500k
        compress
        notifempty
}
```

## SETUP APACHE PYTHON SERVER

Install required aptitude dependencies
```bash
apt-get install 
  apache2 \
  libapache2-mod-wsgi \
  git \
  glances \
  python-dev \
  python-flask \
  python-sqlalchemy \
  python-pip
```

Use pip to install others
```bash
sudo pip install flask virtualenv oauth2client
```

tried to enable mod-wsgi, but looked like it was already enabled
```bash
sudo a2enmod wsgi
```

create the directories for the application
```bash
sudo mkdir /var/www/restaurants
cd /var/www/restaurants
```
clone the repository for the web application
```bash
sudo git clone https://github.com/arizonatribe/apache-python-server-demo.git
```
copy the necessary files from the repository
```bash
sudo cp -r apache-python-server-demo/restaurants .
sudo rm restaurants/README.md
```
remove the repository directory
```bash
sudo rm -r apache-python-server-demo/ 
```

setup virtual environment and install flask and glances into it
```bash
sudo virtualenv restie
source restie/bin/activate
sudo pip install Flask Glances
```

configured monitoring for Glances tool by toggling the RUNNING="false" value
```bash
sudo vim /etc/default/glances
```

created a RestaurantsApp.conf file for apache
```bash
sudo vim /etc/apache2/sites-available/RestaurantsApp.conf
```
Added in WSGIScriptAlias and Alias values for /static and /templates dirs
```xml
<VirtualHost *:80>
                ServerName ec2-52-34-56-206.us-west-2.compute.amazonaws.com
                ServerAdmin student@udacity.com
                WSGIScriptAlias / /var/www/restaurants/restaurants.wsgi
                <Directory /var/www/restaurants/restaurants/>
                        Order allow,deny
                        Allow from all
                </Directory>
                Alias /static /var/www/restaurants/restaurants/static
                <Directory /var/www/restaurants/restaurants/static/>
                        Order allow,deny
                        Allow from all
                </Directory>
                Alias /templates /var/www/restaurants/restaurants/templates
                <Directory /var/www/restaurants/restaurants/templates/>
                        Order allow,deny
                        Allow from all
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
enabled the virtual host
```bash
sudo a2ensite RestaurantsApp
```

created a wsgi file for serving this application
```bash
vim restaurants.wsgi
```

```python
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/restaurants/")

from restaurants import app as application
application.secret_key = 'super_secret_key'
```

went to console.developers.google.com and recreated the api credentials to
include this site as a requesting origin domain and redirect URL, copying
the contents of the JSON file generated by google into the following file:
```bash
vim /var/www/restaurants/restaurants/client_secrets.json
```

restarted apache
```bash
sudo service apache2 restart
```

verified at http://52.34.56.206

## DATABASE SETUP

install postgres tools
```bash
sudo apt-get install postgresql postgresql-contrib
```
logged in as postgres user
```bash
sudo -u postgres psql
```
created the user/role "catalog" and granted role permissions
```sql
CREATE USER catalog WITH LOGIN;
```
created the database for the user
```sql
CREATE DATABASE catalog;
\q;
```
created the OS user "catalog" to match
```bash
sudo adduser catalog
```
verified can login without and with a password
```bash
sudo -u catalog psql
\q;
su - catalog
psql
\q;
```
verified no remote connections allow in postgres config
```bash
sudo vim /etc/postgresql/9.3/main/pg_hba.conf
```

after changing the connection strings in the python files that create the 
tables and seed them with data, ran the python files
```bash
python /var/www/restaurants/restaurants/database_setup.py
python /var/www/restaurants/restaurants/lotsofmenus.py
```

logged in as postgres user and set permissions for the catalog user to just
alter records on these tables
```sql
ALTER DEFAULT PRIVILEGES FOR ROLE catalog IN SCHEMA public 
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO catalog;
```
Also revoked connection to catalog database to anyone but the catalog user
```sql
REVOKE CONNECT ON DATABASE catalog FROM PUBLIC;
GRANT CONNECT ON DATABASE catalog TO catalog;
```
Made sure to grant ability to alter sequences too
```sql
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO catalog;
```

after verifying successful creation of records, removed script
```bash
sudo rm /var/www/restaurants/restaurants/lotsofmenus.py
```

restarted apache
```bash
sudo service apache2 restart
```

verified site is up @ http://ec2-52-34-56-206.us-west-2.compute.amazonaws.com
verified oauth login with Google+ succeeds and am able to create/modify restaurants, menu items, etc.

## Resources used to complete these tasks
* [Using Fail2Ban to protect an Apache Web Server](https://www.digitalocean.com/community/tutorials/how-to-protect-an-apache-server-with-fail2ban-on-ubuntu-14-04)
* [Setting up a Cron job to auto update/upgrade packages](https://help.ubuntu.com/community/AutomaticSecurityUpdates)
* [Setting up NTP on Ubuntu](http://blogging.dragon.org.uk/setting-up-ntp-on-ubuntu-14-04/)
* [Setting up Time Synchronization on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-set-up-time-synchronization-on-ubuntu-12-04)
* [Setting Timezones in Ubuntu](http://unix.stackexchange.com/questions/110522/timezone-setting-in-linux)
* [Deploying a Flask Application on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
* [Revoking public access to Postgres Database and Granting specific privileges](http://dba.stackexchange.com/questions/33943/granting-access-to-all-tables-for-a-user)
* [SqlAlchemy connection strings for Postgres](http://docs.sqlalchemy.org/en/latest/core/engines.html)
* [Managing Roles and Grant permissions in Postgres](https://www.digitalocean.com/community/tutorials/how-to-use-roles-and-manage-grant-permissions-in-postgresql-on-a-vps--2)
* [Securing Postgres on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
* [Configuring UFW Firewall settings on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-setup-a-firewall-with-ufw-on-an-ubuntu-and-debian-cloud-server)
* [Configuring application monitoring with Glances](http://www.cyberciti.biz/faq/linux-install-glances-monitoring-tool/)


