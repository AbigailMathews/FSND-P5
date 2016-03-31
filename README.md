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
Public IP address: 52.26.87.80

54.208.153.214 --
ec2-54-208-153-214.compute-1.amazonaws.com

54.209.247.31 --
ec2-54-209-247-31.compute-1.amazonaws.com

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
grader ALL=(ALL) NOPASSWD: ALL  ... Change to grader ALL=(ALL:ALL) ALL

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

**********
### Part 2: Setting up Apache and mod-wsgi

Our application works on python v 2.7.6, so we'll install a virtual environment from which
to run our program.

*Upgrade python from version 2.7.6 to version 2.7.9*

`sudo apt-get install python-pip`

get the newest python tar file:
```
sudo apt-get install build-essential
sudo apt-get install libreadline-gplv2-dev libncursesw5-dev libssl-dev libsqlite3-dev tk-dev libgdbm-dev libc6-dev libbz2-dev

sudo wget https://www.python.org/ftp/python/2.7.9/Python-2.7.9.tgz
sudo tar -xvzf Python-2.7.9.tgz
cd Python-2.7.9
```
*****
./configure --prefix /usr/local/lib/python-2.7.9 --enable-ipv6 
(if using virtualenv)
*****
```
sudo ./configure
sudo make
sudo make install
```

* lots of messy stuff that's not working around here...

**********

*Install Apache web server, wsgi*

`sudo apt-get install apache2`

You will now be able to see the Apache It Works! splash page if you navigate your browser
to your server's IP Address
```
sudo apt-get install libapache2-mod-wsgi python-dev
sudo a2enmod wsgi
```
restart apache:

`sudo service apache2 restart`

************

### Part 3: Clone our application, install dependencies

*Download Git*

`sudo apt-get install git`


Clone our directory from github into /var/www:
```
sudo git clone https://github.com/AbigailMathews/catalog_archive.git catalog
cd catalog
```
Now when we create our virtual environment, we will use the version of python 
we just installed (2.7.9)

`sudo virtualenv -p /usr/local/lib/python2.7.9/bin/python venv`

I discovered that this venv directory was created with no permissions, so I changed
them so that I could launch the environment.
```
sudo chmod -R 777  venv
source venv/bin/activate
```
You are now working within the virtual environment, indicated by the (venv) at the beginning
of the prompt.

*Install Dependencies*

Now we can install Flask as well as any other packages our application relies upon.
```
sudo pip install Flask

sudo apt-get install libpq-dev
pip install psycopg2
```
// install other dependencies //


### Part 4: Configure Apache

*Create a .htaccess file*

In /var/www/catalog:
```
sudo touch .htaccess
sudo nano .htaccess
```
Add the line: `RedirectMatch 404 /\.git`

`sudo nano /etc/apache2/sites-available/catalog.conf`

Paste the following text into the .conf file:
```
 <VirtualHost *:80>
      ServerName 52.26.87.80
      ServerAdmin admin@52.26.87.80
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
  
`sudo service apache2 reload`

`sudo a2ensite catalog` (sudo a2dissite 000-default)



*Configure WSGI File*

Still in /var/www/catalog
`sudo nano catalog.wsgi`

(missing bit)

USE activate_this.py if using virtualenv


### Part 6: Set up Postgresql
*Install postgresql*

```
apt-get install postgresql
sudo nano /etc/postgresql/9.3/main/pg_hba.conf
sudo passwd postgres
```
create a temporary password to work as the postgres user
```
su postgres
psql
```
In the psql prompt:

<< OR CREATE USER >>
```
>> CREATE ROLE catalog;
>> ALTER USER catalog WITH PASSWORD 'catalog';
>> ALTER ROLE catalog WITH LOGIN;
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

We can now run `database_setup.py` and then `db_populate.py` to create our initial 
database state.

### Part 7: Launch

Um, yeah

---

## Sources

A list of sites visited, forum posts, etc. to be copied from my bookmarks folder, with links and possible comments.
