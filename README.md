# Linux Server Configuration
### Project 5 -- Udacity Fullstack Nanodegree


## Background
The purpose of this project is to deploy a Flask (python) application on the
Amazon Web Service EC2 platform. Furthermore, image files are set up to host from
AWS S3 service. This project was completed as Project 5 of the Udacity Fullstack
Nanodegree program.

The project being hosted is an outgrowth of Project 3 - Create a Catalog application.
In this case, the 'catalog' is a personal archive platform, which allows users to 
save images and other documents with associated metadata.

---

## Accessing the Server

IP Address: 52.90.89.207
[http://ec2-52-90-89-207.compute-1.amazonaws.com](http://ec2-52-90-89-207.compute-1.amazonaws.com)

The user grader is provided, along with a keypair for ssh access to the server. The application
itself can be viewed by navigating to the above address.

---

## Project Steps
This section outlines the major steps undertaken in setting up the AWS instance.

### Part 1: Users and ssh Configuration
*Create New User, grader, with sudo access*

Our first step is to configure a new user, 'grader' to replace the default login, as well
as to disable root access. The grader user is granted sudo access. The following commands
are executed to achieve these ends:

`sudo adduser grader`

during this process, when asked to assign a password for grader, do so -- it will be used
for sudo access

`sudo nano /etc/sudoers.d/grader`

Add the line:

`grader ALL=(ALL:ALL) ALL`

The grader user is also assigned a unique keypair for ssh access.

`sudo mkdir /home/grader/.ssh`

the public key for this user is then copied into the file `/home/grader/.ssh/authorized_keys`


Once grader is properly configured, we remain logged in as grader to continue our work.

*Update Packages*

First we update/upgrade all installed packages

``` 
sudo apt-get update 
sudo apt-get upgrade
```
Now we change the port for ssh to 2200 from 22, and disable both root login and password
authentication (the default). This means that our access to the server will be through the 
grader user and the keypair previously generated for this user.
```
sudo nano /etc/ssh/sshd_config
sudo service ssh restart
```
*Firewall*

Now we would like to set up a firewall. We use UFW (Uncomplicated Firewall)

But! We must also be sure that our server is permitting connections on port 2200, or we
will be locked out of our ssh connection. We also want to permit connections on 
the other ports we intend to use (80 and 123). So we configure our UFW firewall.
```
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 2200
sudo ufw allow 2200/tcp

sudo ufw allow 80
sudo ufw allow 80/tcp

sudo ufw allow 123
sudo ufw allow 123/tcp
```
Check what you have done:

`sudo less /lib/ufw/user.rules`

Before (cross your fingers!):

`sudo ufw enable`

You can now check on your firewall with `sudo ufw status`


*Configure date, changing to UTC if necessary*

`date` will tell you your current timezone. In the case of this configuration, the server was
already set up with a UTC time zone. However, you can change the timezone if necessary
with the following command: `sudo dpkg-reconfigure tzdata`
A menu-based tool will allow you to change the timezone.


### Part 2: Setting up Apache and mod-wsgi

*Install Apache web server, and mod-wsgi*

`sudo apt-get install apache2`

You will now be able to see the Apache It Works! splash page if you navigate your browser
to your server's IP Address
```
sudo apt-get install libapache2-mod-wsgi python-dev
sudo a2enmod wsgi
```
restart apache:

`sudo service apache2 restart`


### Part 3: Clone our application, install dependencies

*Download Git*

`sudo apt-get install git`


Clone our directory from github into /var/www:
```
sudo git clone https://github.com/AbigailMathews/catalog_archive.git catalog
cd catalog
```
Change the name of the app.py file to __init__.py, and change the final lines to disable
debug mode and let the Apache service take over the hosting:

```
if __name__ == '__main__':
    app.run()
```

*Install Dependencies*

Now we can install Flask as well as any other packages our application relies upon.
```
sudo apt-get install python-pip

sudo pip install Flask
sudo pip install Flask-Bootstrap
sudo pip install Flask-SQLAlchemy
sudo pip install Flask-SeaSurf
sudo pip install oauth2client

sudo apt-get install libjpeg8-dev
sudo apt-get install zlib1g-dev

sudo pip install Pillow

sudo apt-get install libpq-dev
sudo pip install psycopg2
```


### Part 4: Configure Apache

*Create a .htaccess file*

In /var/www/catalog:

`sudo nano .htaccess`

Add the line: `RedirectMatch 404 /\.git`

`sudo nano /etc/apache2/sites-available/catalog.conf`

Paste the following text into the .conf file:
```
 <VirtualHost *:80>
      ServerName 52.90.89.207
      ServerAdmin abbymathews@gmail.com
      WSGIScriptAlias / /var/www/catalog/catalog.wsgi
      <Directory /var/www/catalog/catalog/>
          Order allow,deny
          Allow from all
      </Directory>
      Alias /static /var/www/catalog/catalog/static
      <Directory /var/www/catalog/catalog/static/>
          Order allow,deny
          Allow from all
      </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>
```
Now disable the default site, (the Apache It Works! page), and enable the catalog application.
```
sudo a2dissite 000-default
sudo a2ensite catalog
sudo service apache2 reload
```

*Configure WSGI File*

Still in /var/www/catalog:
`sudo nano catalog.wsgi`

Copy the following code into the file:

```
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.append('/var/www/catalog')
sys.path.append('/var/www/catalog/catalog')

from catalog import app as application
application.secret_key = 'my_secret_key'
```

### Part 6: Set up Postgresql
*Install postgresql*

```
sudo apt-get install postgresql
sudo nano /etc/postgresql/9.3/main/pg_hba.conf
sudo passwd postgres
```
Create a password to work as the postgres user

*Create catalog DB, Create catalog postgres user*

```
su postgres
psql
```
In the psql prompt:

```
>> CREATE USER catalog;
>> ALTER USER catalog WITH PASSWORD 'catalog';
>> CREATE DATABASE catalog WITH OWNER catalog;
>> \c catalog
>> REVOKE ALL ON SCHEMA public FROM public;
>> GRANT ALL ON SCHEMA public TO catalog;
>> \q
```
Quitting psql, log on as grader again, and restart postgresql:
```
su grader
sudo service postgresql restart
```
Change the application's DB calls to reflect the new catalog DB name.
For instance, create engine statements should now read:
`engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`

In /var/www/catalog/catalog, we can now run `database_setup.py` and then `db_populate.py` 
to create our initial database state.

### Part 7: Check our OAuth setup

The catalog site provides two options for users to log in to the site using Oauth. We
need to make some changes to ensure they continue to function properly.

First we must copy over the client_secrets.json and fb_client_secrets.json, which are
not provided in the github repo.

A couple of updates to our __init__.py are necessary to change the paths of 
client_secrets.json and fb_client_secrets.json to the full path, rather than the relative
paths. These paths should appear as `/var/www/catalog/catalog/client_secrets.json`
and `/var/www/catalog/catalog/fb_client_secrets.json` respectively.

Finally, we visit console.developers.google.com and developers.facebook.com/apps 
For Google, update the 'Authorized JavaScript origins' to include the URL for the site,
in this case `http://ec2-52-90-89-207.compute-1.amazonaws.com`. In 'Authorized redirect URIs,'
add entries for `http://ec2-52-90-89-207.compute-1.amazonaws.com` + '/login' '/gconnect' and 
'/collections'

For Facebook, it is sufficient to add `http://ec2-52-90-89-207.compute-1.amazonaws.com` to
the Settings --> Basic Site URL field.

### Part 8: Launch

Do a final `sudo service apache2 restart`, and check the site with a browser. If
everything has been set up properly, you will be able to view and interact with your site.

Otherwise, debug by referencing `/var/log/apache2/error.log`

---

### Extra Features
*Cron jobs*

Set up the root crontab to automatically manage the package update/upgrade process.
`sudo crontab -e`
to access the root crontab, and then create the following rules:
```
@daily apt-get update >> /var/log/cron-updates.log 2>&1
15 0 * * * apt-get upgrade >> /var/log/cron-upgrades.log 2>&1

# Cycle out the logs each week

30 0 * * 0 mv /var/log/cron-updates.log /var/log/cron-updates-last.log
30 0 * * 0 mv /var/log/cron-upgrades.log /var/log/cron-upgrades-last.log

45 0 * * 0 rm -f /var/log/cron-updates.log
45 0 * * 0 rm -f /var/log/cron-upgrades.log
```
---
*Glances*

Glances is a system monitoring tool, which allows a comprehensive snapshot view of
your system status.

Install it:
`sudo pip install glances`

Run it:
`glances`

[View of the Glances system status window](https://github.com/AbigailMathews/FSND-P5/blob/master/glances.png)
---
*Fail2Ban*

Fail2Ban is a service that can protect a system that is being targeted by brute force
attacks or other botting threats. We will configure Fail2Ban to protect our ssh
connection, as well as Apache itself.

Installation:
```
sudo apt-get update
sudo apt-get install fail2ban
```
We are going to be working with Fail2Ban's jails to modify the default rules and add some
of our own. Let's first make a new copy of the jail config:
`sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`
and open it: `sudo nano /etc/fail2ban/jail.local`

For this configuration, I increased the bantime to 1800 sec (from 600), and increased 
maxretry to 5 (from 3).

Further, set up email alerts by putting relevant data into the destemail, sendername,
and mta, and changing the action to read `action = $(action_mwl)s

Under the general settings, each service has it's own jail, with specific rules. We are
concerned with the ssh and apache-related rules.

For ssh, we only modify the port rule, changing it to 2200:
```
[ssh]

enabled  = true
port     = 2200
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 6
```

For Apache, we enable `apache`, `apache-overflows` and created new `apache-badbots` and
`apache-nohome` rules replicating the apache-overflows model.. More details on these 
rules can be found in the Digital Ocean guide on this topic.

If we want Fail2Ban to correctly send emails when action is taken, we must also 
`sudo apt-get install sendmail`

There are a number of other configuration options with Fail2Ban, and it may be fruitful
to review these by reviewing the DO guides or the documentation.

## Sources
Thanks to students on the forums, who suggested both Fail2Ban and Glances, as well as 
provided links to the very helpful Digital Ocean tutorials, which I found extremely helpful.
* Norbert Stuken's repo: [https://github.com/stueken/FSND-P5_Linux-Server-Configuration](https://github.com/stueken/FSND-P5_Linux-Server-Configuration)
* UFW: [https://www.digitalocean.com/community/tutorials/how-to-setup-a-firewall-with-ufw-on-an-ubuntu-and-debian-cloud-server](https://www.digitalocean.com/community/tutorials/how-to-setup-a-firewall-with-ufw-on-an-ubuntu-and-debian-cloud-server)
* Apache/mod-wsgi: [http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/](http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/)
* Postgresql: [http://killtheyak.com/use-postgresql-with-django-flask/](http://killtheyak.com/use-postgresql-with-django-flask/)
* Cron Jobs: [http://kvz.io/blog/2007/07/29/schedule-tasks-on-linux-using-crontab/](http://kvz.io/blog/2007/07/29/schedule-tasks-on-linux-using-crontab/)
* Glances Documentation: [https://pypi.python.org/pypi/Glances](https://pypi.python.org/pypi/Glances)
* Two Great Fail2Ban Guides from Digital Ocean:
    - [https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)
    - [https://www.digitalocean.com/community/tutorials/how-to-protect-an-apache-server-with-fail2ban-on-ubuntu-14-04](https://www.digitalocean.com/community/tutorials/how-to-protect-an-apache-server-with-fail2ban-on-ubuntu-14-04)
    
## Future Improvements
It was my intention to use virtualenv to create an isolated environment in which to run
the Flask app. However, despite much assistance from other students in the forums and Slack
channel, in particular Crewe, I was unable to resolve a problem where mod-wsgi was not using 
the correct version of Python.
    
