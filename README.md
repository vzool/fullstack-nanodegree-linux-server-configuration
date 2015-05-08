# Fullstack Nanodegree Linux Server Configuration

Really I did configure a [Linux][5] server to run a [WSGI][2] [Flask][3] application, I have some knowledge about it in [PHP][4]. But these stuff make me know what is The Standards to serve applications online in [Python][6].

I made it by about 11 steps and I organized my solution as Tasks collection to make things easier to track and point.

## Task #0 Update packages and Change timezone to UTC

To update all currently installed packages:

```shell
sudo apt-get update
sudo apt-get upgrade
```

To configure the local timezone to UTC:

```shell
sudo dpkg-reconfigure tzdata
```

Then select `None of the above`, after that you will find UTC in the bottom of the next page.

Finally, any VPS need its own hostname by overwrite what inside in `/etc/hostname` and just add your name, so I changed mine into:

```shell
sudo echo udacity.vzool.me > /etc/hostname
```

## Task #1 Change SSH Server port(2200) and Enable Firewall to accept(2200,80,123)

When you want to change your default SSH Server port from 22 to any other one you need to edit and change its config file which located at `/etc/ssh/sshd_config` and remember it's `sshd_config` not `ssh_config`.

```shell
sudo nano /etc/ssh/sshd_config
```

Then change Port:

```
# Package generated configuration file
# See the sshd_config(5) manpage for details

# What ports, IPs and protocols we listen for
Port 2200
```

Save the file and restart SSH Server:

```shell
sudo service ssh restart
```

Finally, to enable firewall and accept only specific ports:

```shell
# Enable Firewall
sudo ufw enable

# Add Rule for port 2200
sudo ufw allow 2200

# Add Rule for port 80
sudo ufw allow 80

# Add Rule for port 123
sudo ufw allow 123
```


## Task #2 Create a new user(grader) with sudo permission

First I need to launch my Virtual Machine(VPS), and then login.

Some commands needs to execute:

```sell
# Add a new user
sudo useradd -m grader
```

Then add sudo permission to grader, and that's done by this:

```shell
# Remember don't change >> to > which will overwrite anything in sudoers file.
sudo echo "grader  ALL=(ALL:ALL) ALL" >> /etc/sudoers
```

Finall, you will need to set a password for grader by:

```shell
sudo passwd grader
```


## Task #3 Install PostgreSQL, create database(grader) for user(grader)

```shell
# install PostgreSQL
sudo apt-get install postgresql

# login as postgres in linux terminal
su postgres

# login to postgresql shell
psql

# inside psql create database

psql> CREATE DATABASE catalog;

# create user catalog

psql> CREATE USER catalog;

# give user full permission under catalog database

psql> GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;

# Finally, set password for user catalog

psql> ALTER USER catalog PASSWORD '<Select-Your-Own-Password>';

# Quit from postgresql shell

psql> \q

```

## Task #4 Install and configure Apache to serve a Python mod_wsgi application

```shell
# install apache and prerequisites
sudo aptitude install apache2 apache2.2-common apache2-mpm-prefork apache2-utils libexpat1 ssl-cert

# install mod_wsgi apache module
sudo aptitude install libapache2-mod-wsgi

# restart apache
sudo /etc/init.d/apache2 restart
```

### mod_wsgi Installation: Building and installing mod_wsgi from source

Remember, always looking for the latest versions at `https://code.google.com/p/modwsgi/`.

```shell
wget https://github.com/GrahamDumpleton/mod_wsgi/archive/3.5.tar.gz
tar xvfz 3.5.11.tar.gz
```

### Build mod_wsgi and install it

```shell
cd mod_wsgi-3.5
./configure
make
sudo make install
```

### Make mod_wsgi module to be loaded by apache2

```shell
# Add mod_wsgi to Apache2 modules
sudo echo "LoadModule wsgi_module /usr/lib/apache2/modules/mod_wsgi.so" > /etc/apache2/mods-available/mod-wsgi.load

# enable it
sudo a2enmod mod-wsgi

# restart apache2
sudo service apache2 restart
```

## Task #5 Install Catalog App

```shell
# First install git
sudo apt-get install git

# Then clone my app from GitHub repository.
sudo git clone https://github.com/vzool/fullstack-nanodegree-item-catalog /var/www/catalog

# make wsgi directory inside catalog
sudo mkdir /var/www/catalog/wsgi

# create a `flask.wsgi` inside `wsgi` directory
sudo touch /var/www/catalog/wsgi/flask.wsgi

# create `__init__.py` file inside `wsgi` directory
sudo touch /var/www/catalog/wsgi/__init__.py

# change file permissions
sudo chmod 755 /var/www/catalog -R
```
### Install Application Dependencies

```shell
# install pip
sudo apt-get install python-pip

# install httplib2
sudo pip install httplib2

# install oauth2client
sudo pip install oauth2client

# install flask
sudo pip install oauth2client

# install sqlalchemy
sudo pip install sqlalchemy

# install psycopg2
sudo pip install psycopg2

# install requests
sudo pip install requests
```

### Configure Database

First open `database_setup.py` and Modify line 73, and open `app.py` and Modify line 25 which both of them will look like the following line:

```
engine = create_engine('sqlite:///catalog_item.db')
```

To

```
engine = create_engine('postgresql://catalog:<Select-Your-Own-Password>@localhost:5432/catalog')
```

Change <Select-Your-Own-Password> with your own password which you ahev set in psql for catalog user.

After that fire this script `database_setup.py` to build tables inside catalog database

```
sudo python database_setup.py
```

### Configure `app.py`

Change value for `client_secret_file` at line 13 which will be look like this line:

```
client_secret_file = 'client_secret_404731601094-8lv7aqee2gdp3vun964is8p9p0csgasn.apps.googleusercontent.com.json'
```

To new value

```
client_secret_file = '/var/www/catalog/client_secret_404731601094-8lv7aqee2gdp3vun964is8p9p0csgasn.apps.googleusercontent.com.json'
```

Then copy line 474 after line 11

```
app.secret_key = b'\xc2q;\x15z\t\xafT&\xfd\x81\x93\xfc^\xb9\xa0\xf4\xa6\x93\xc2\xe9\xd8\x94\x1d'
```



### Configure `wsgi/flask.wsgi`

Open it with your favorite editor and paste the following codez:

```sell
import os
import sys
import logging
logging.basicConfig(stream=sys.stderr)

##Replace the standard out
sys.stdout = sys.stderr

##Add this file path to sys.path in order to import settings
sys.path.insert(0, os.path.join(os.path.dirname(os.path.realpath(__file__)), '../..'))

##Add this file path to sys.path in order to import app
sys.path.append('/var/www/catalog/')

##Create appilcation for our app
from app import app as application
```

### Configure apache2

Paste following lines inside `/etc/apache2/sites-available/000-default.conf`

```shell
ServerName udacity.vzool.me

ServerAdmin aeemh.sdn@gmail.com

#DocumentRoot /var/www/html

WSGIDaemonProcess catalogflask user=www-data group=www-data threads=5
WSGIScriptAlias / /var/www/catalog/wsgi/flask.wsgi
WSGIScriptReloading On

<Directory /var/www/catalog/wsgi>
	Order allow,deny
	Allow from all
</Directory>
```


Restart apache2

```
sudo service apache2 restart
```

# Test The Application

Open your favorite browser with one of these links:

- [http://52.24.14.78/][8]
- [http://udacity.vzool.me/][9]

# Licence

It's Completely Free. But, Do whatever you like to do on your own full responsibility;

This licence is known with [MIT License][1] in professional networks.

[1]: http://vzool.mit-license.org
[2]: https://code.google.com/p/modwsgi/
[3]: http://flask.pocoo.org/
[4]: http://php.net/
[5]: https://www.linux.com/
[6]: https://www.python.org/
[7]: https://github.com
[8]: http://52.24.14.78/
[9]: http://udacity.vzool.me/
