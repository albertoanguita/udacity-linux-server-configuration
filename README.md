# Linux Server Configuration - Full Stack Web Developer Nanodegree project

This document describes all steps performed and all configuration changes realized to achieve the set up of the linxu server for project 5.

IP address: 52.26.114.117
SSH port: 2200

URL to catalog application:

http://52.26.114.117/

http://ec2-52-26-114-117.us-west-2.compute.amazonaws.com/

## User configuration

After loging in for the first time with root credentials, a user named "grader" was created. The command was:

`adduser grader`

The password 'grader' was set for this user.

The public key was copied from the root account to the new users, placing the authorized_keys file in the .ssh directory.

Then, the file /etc/sudoers was edited to give the new user access to sudo commands. The following line was added:

`grader  ALL=(ALL:ALL) ALL`

## Package updating

All packages in the machine were updated, using the following commands:

```
sudo apt-get update
sudo apt-get upgrade
```

## SSH port change

The SSH listening port was changed from 22 to 2200. The file /etc/ssh/sshd_config was edited, and the line 

`Port 22`

Was changed for 

`Port 2200`

I also ensured that the line `PasswordAuthentication no` was present, to avoid password-based login, and included the line `PermitRootLogin no` to disable root login.

## Firewall configuration

The following commands were employed to properly setup the firewall:

```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow http
sudo ufw allow ntp
sudo ufw enable
```

## Time zone configuration

My machine was already configured in UTC (I run the command `date` to check the time) so no configuration changes were needed.

## Apache installation and configuration

Next step was installing apache and the wsgi mod for communicating with my Python application. I ran the following commands:

```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi
```

Then I configured the apache server to handle incoming connections through wsgi. I edited the file /etc/apache2/sites-enabled/000-default.conf, and included the following line:

`WSGIScriptAlias / /var/www/html/itemcatalog/udacity-item-catalog/catalog.wsgi`

Since that would be the wsgi file giving access to my application. Also, I configured what files under the docroot would be directly accessible, providing access only to the wsgi file:

```
<Directory /var/www/html/itemcatalog/udacity-item-catalog/>
        order deny,allow
        Deny From All
        <Files catalog.wsgi>
                Order allow,deny
                Allow from ALL
        </Files>
</Directory>
```

And finally I restarted the apache service, so changes were loaded:

`sudo service apache2 restart`

## PostgreSQL installation and configuration

I installed the necessary packages for postgres:

```
sudo apt-get install postgresql
sudo apt-get install postgresql-contrib
```

I denied all external connections to postgress, by editing the file /etc/postgresql/9.3/main/postgresql.conf, and included the line:

`listen_addresses = 'localhost'`

So the server refuses remote connections. I restarted the server using `sudo /etc/init.d/postgresql restart`.

After this, I created the database for my application. First, I connected with the default user 'postgres' and gave it a password:

`sudo -u postgres psql postgres`

Then, I connected to the postgres shell with the postgres user:

`sudo psql -h 127.0.0.1 -U postgres`

and issued the command for creating a new database, called 'catalog':

`CREATE DATABASE catalog;`

Then I created a new user called 'catalog' with password 'grader':

`CREATE USER catalog;`

And granted this new user access to the catalog database:

`GRANT CONNECT ON DATABASE catalog TO catalog;`

I tested that the new user could create and drop tables, input new data, but not drop the entire database. 

## Python packages

I installed the required packages for Python: Flask, SQLAlchemy and oauth2client

```
pip install Flask
pip install SQLAlchemy
pip install oauth2
```

## Catalog app

I installed git (`sudo apt-get install git`) and cloned my catalog app repository. The main app files were located in `/var/www/html/itemcatalog/udacity-item-catalog/itemcatalog`.

I edited my database_setup.py and database_populator.py so they connect to PostgreSQL instead of SQLite. I changed this:

`engine = create_engine('sqlite:///game_catalog.db')`

for this:

`engine = create_engine('postgresql://catalog:grader@localhost:5432/catalog')`

And executed them one after another. The database was properly set up and populated.

I created the wsgi file at `/var/www/html/itemcatalog/udacity-item-catalog/catalog.wsgi`. The contents of the file are:

```
import sys
sys.path.insert(0, '/var/www/html/itemcatalog/udacity-item-catalog')
sys.path.insert(0, '/var/www/html/itemcatalog/udacity-item-catalog/itemcatalog')

from itemcatalog import app as application
```

and changed its access permissions to only read:

`sudo chmod 444 catalog.wsgi`

I also had to edit the views.py file to connect to postgres and to properly locate the client_secrets.json file (I used a full path). 

I logged to my google developer console and added the FQDN of my app (http://ec2-52-26-114-117.us-west-2.compute.amazonaws.com) to the list of authorized javascript origins. I found the FQDN using a reverse DNS service.

I verified I could not access any other file in the project.

I restarted the apache server again, and the application fully worked.

