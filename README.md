# Linux-Server-Configuration
Project #5 for the Udacity Full Stack Web Developer Nanodegree

### About

For the final	 Udacity nanodegree project, I configured a linux Amazon EC2 instance and deployed my catalog app from [project #3](https://github.com/sgreenlee/Item-Catalog). The application can be found [here](http://ec2-52-88-115-228.us-west-2.compute.amazonaws.com/).

### Configuration details

Here are the steps I took to secure the server and get my application running.

#####Upgrading software

After logging in to the server, the first thing I did was update all installed software packages:
```
$ apt-get update
$ apt-get upgrade
```

#####Creating new users and configuring ssh

My next order of business was to create the appropriate users, give them sudo privi 	leges and allow them to ssh into the server. First, I added the new users:

```
$ adduser {newuser}
```

Then for each new user I created a file in the `sudoers.d` directory: `$ sudo nano /etc/sudoers.d/{newuser}` with the following text:
```
{newuser} ALL=(ALL) NOPASSWD:ALL
```
```
$ mkdir /home/{newuser}/.ssh
$ touch /home/{newuser}/.shh/authorized_keys
$ chown {newuser} /home/{newuser}/.ssh
$ chown {newuser} /home/{newuser}/.ssh/authorized_keys
$ chmod 700 /home/{newuser}/.ssh
$ chmod 600 /home/{newuser}/.ssh/authorized_keys
```

Then I added public keys into each `authorized_keys` file. Finally, I opened the `/etc/ssh/sshd_conig` file and set the following configuration settings:

```
Port 2200
...
PermitRootLogin no
...
PasswordAuthentication no
```

After this I exited the root login session and logged back into the server with my new user account and public key.

#####Setting the time zone

To set the time zone I ran the command:

```
$ sudo dpkg-reconfigure tzdata
```

This opened a dialog. From the menu I selected 'None of the above' and then 'UTC.'

#####Setting up a firewall

Next I configured a firewall to only allow incoming connections on ports 80, 123 and 2200:
```bash
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 2200/tcp
$ sudo ufw allow www
$ sudo ufw allow ntp
$ sudo ufw enable
```

##### Installing and configuring fail2ban

To protect against brute force attacks I installed fail2ban, a software package that blocks ip addresses with multiple failed login attempts within a certain amount of time. I installed fail2ban from apt-get:

```
$ sudo apt-get install fail2ban
```

Configuring fail2ban was simple, as the default configuration settings were mostly adequate. All I had to do was copy the default configuration file to new file to prevent updates from overwriting my configuration settings:

```
$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Then, within the new `jail.local` file I changed the ssh port from 22 to 2200, saved the file and restarted fail2ban: 
```
$ sudo service fail2ban restart
```


#####Installing and configuring Apache and mod-wsgi

Next I installed git, apache and mod-wsgi:

```
$ sudo apt-get install apache2 libapache2-mod-wsgi git
```

and ran the following command to enable mod-wsgi:
```
$ sudo a2enmod wsgi
```

Then, in the `/var/www/` directory, I cloned my project into the directory `Item-Catalog`, where I created the file `application.wsgi` with the following contents:	

```python
#!/usr/bin/python

import sys
import os
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/Item-Catalog/")

# create environment variables from environment.sh
env = open('/var/www/Item-Catalog/environment.sh', 'r')
for line in env:
        var, val = line.split('=')
        os.environ[var] = val[:-1]
env.close()

from itemcatalog import app as application
```

The application's configuration file reads sensitive information from environment variables. `applications.wsgi` writes these variables using a file named `environment.sh` in the Item-Catalog directory, so I created `environment.sh` and set all the required variables to appropriate values. Now the application has the following directory structure:
```
- Item-Catalog\
----- application.wsgi
----- environment.sh
----- ...
----- itemcatalog\ # application package
--------- ...
```

Next, I created a new virtual host. I created the file `/etc/apache2/sites-available/CatalogApp.conf` and added the folowing contents to it:
```
<VirtualHost *:80>
                ServerName 52.88.226.136
                ServerAdmin sam.a.greenlee@gmail.com
                WSGIScriptAlias / /var/www/Item-Catalog/application.wsgi
                <Directory /var/www/catalog/catalog/>
                        Order allow,deny
                        Allow from all
                </Directory>
                Alias /static /var/www/catalog/catalog/static
                <Directory /var/www/catalog/catalog/static/>
                        DirectoryIndex /notfound
                        FallbackResource /notfound
                        Order allow,deny
                        Allow from all
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Finally, I enabled the new virtual host and restarted apache:

```
$ sudo a2ensite ConfApp
$ sudo service apache2 restart
```

#####Installing and configuring Postgresql

First I installed Postgresql: `$ sudo apt-get install postgresql postgresql-contrib`

Next I checked the configuration file to make sure that remote connections were not allowed:
```
$ sudo /etc/postgresql/9.1/main/pg_hba.conf
```

No changes were necessary here. Then I created a linux user named catalog, a postgresql role with the same name, a database named catalog_app and gave the catalog role permissions to the catalog_app database:

```text
$ sudo adduser catalog
$ sudo passwd catalog
...{set catalog password}...
$ sudo su postgres
$ psql
  CREATE USER catalog; WITH PASSWORD '{password}'
  CREATE DATABASE catalog_app;
  \c catalog_app;
  REVOKE ALL ON SCHEMA public FROM public;
  GRANT ALL ON SCHEMA public TO catalog;
  \q
$ exit
```

##### Installing python requirements and final adjustments

Next I installed the python packages my application needs to run using my requirements.txt file:
```
$ sudo apt-get install python-pip python-psycopg2
$ sudo pip install -r /var/www/catalog/catalog/requirements.txt
```

Since my application needs to write images to the `/var/www/catalog/catalog/static/images/' directory, I changed the owner of this directory to the user that apache runs as:
```
$ sudo chown www-data /var/www/catalog/static/images
```

Finally, I copied my Google and Facebook credentials files into the project directory and added my project url to the approved redirect urls and javascript origins on my Facebook and Google developer consoles.

#####Configuring mod_status

I used the apache module mod_status to monitor the performance and availability of the application. By default, mod_status was already installed and enable on my version of apache, so the only changes I had to make were adding these lines inside my virtual host configuration:

```
<Location /server-status>
    SetHandler server-status
    Order deny,allow
    Deny from all
    Allow from localhost 
</Location>
```

and restarting apache: `$ sudo service apache2 restart`

In order to view mod_status' reports I installed lynx, a terminal-based web browser:

```
$ sudo apt-get install lynx
```
After which I could view the reports with the command:
```
$ lynx http://localhost/server-status
```

#####Setting up automatic security updates

I enabled automatic security upgrades with the following commands
```
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

and selected yes in the dialog that appeared.

### Sources

In addition to the Configuring Linux Web Servers Udacity course, I relied on several third-party resources to complete this project. Here they are:

 - [Setting the ssh port](http://ubuntuforums.org/showthread.php?t=1591681)
 - [Changing the timezone](https://help.ubuntu.com/community/UbuntuTime#Using_the_Command_Line_.28terminal.29)
 - [Installing and configuring fail2ban](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)
 - [Configuring Postgresql](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
 - [Configuring a flask application with Apache2 and mod-wsgi](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
 - [Monitoring apache2 with mod_status](http://www.tecmint.com/monitor-apache-web-server-load-and-page-statistics/)
 - [Setting file permissions for apache](http://serverfault.com/questions/125865/finding-out-what-user-apache-is-running-as)
 - [Setting up automatic security updates on Ubuntu](http://askubuntu.com/questions/9/how-do-i-enable-automatic-updates)


### Contact

For comments or questions, reach me at [sam.a.greenlee@gmail.com](mailto:sam.a.greenlee@gmail.com)