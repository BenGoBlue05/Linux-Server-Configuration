# Linux Server Configuration
Udacity FSND final project

http://ec2-18-222-114-3.us-east-2.compute.amazonaws.com

## Get Server

#### Create Server Instance
* Create an Ubuntu Linux (18.04) LTS Server on [Amazon Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home/instances)

#### Connect to Instance with SSH
* Download ssh key from Amazon Lightsail using link https://lightsail.aws.amazon.com/ls/webapp/account/keys

* Move `LightsailDefaultPrivateKey-*.pem` into local folder `~/.ssh` and rename it `item_catalog_key.pem`

* Connect to instance (IP 18.222.114.3) from local terminal with
```    
chmod 600 ~/.ssh/item_catalog_key.pem     
ssh -i ~/.ssh/item_catalog_key.pem ubuntu@18.222.114.3
```

## Secure Server
#### Update installed packages   

```
sudo apt-get update
sudo apt-get upgrade
``` 

#### Change SSH port from 22 to 2200
* Edit config file: `sudo nano /etc/ssh/sshd_config`

* Go to line with `#Port 22` uncomment and change it to `Port 2200`

#### Configure Uncomplicated Firewall (UFW)
* Logout from AWS CLI

* Configure firewall ports on Amazon Lightsail website to only allow incoming connections for SSH (port 2200), 
HTTP (port 80), and NTP (port 123)
![alt text](image/firewallconfig.png "Firewall Config Screenshot")

* Reconnect to CLI: 
```
ssh -i .ssh/item_catalog_key.pem -p 2200 ubuntu@18.222.114.3`
```

* Turn on firewall: `sudo ufw enable`

* Configure ports:

```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow 123/udp
sudo ufw deny 22
```

* Confirm firewall correctly configured: `sudo ufw status`   
This should output the following:

```
Status: active

To                         Action      From
--                         ------      ----
2200/tcp                   ALLOW       Anywhere                  
80/tcp                     ALLOW       Anywhere                  
123/udp                    ALLOW       Anywhere                  
22                         DENY        Anywhere                  
2200/tcp (v6)              ALLOW       Anywhere (v6)             
80/tcp (v6)                ALLOW       Anywhere (v6)             
123/udp (v6)               ALLOW       Anywhere (v6)             
22 (v6)                    DENY        Anywhere (v6)    
```



## Give Grader Access

#### Create grader as new user
`sudo adduser grader`

#### Give grader permission to sudo
* Edit subdoers file: `sudo visudo`

* Add `grader  ALL=(ALL:ALL) ALL` in the privileges specification section

#### Generate SSH key pair for grader
* Logout from connection to remote server

* Run `ssh-keygen` in local machine to generate key pair `~/.ssh/grader_key` and `~/.ssh/grader_key.pub`

* View public key `cat ~/.ssh/grader_key.pub` and copy

* Login to grader's VM and create file `.ssh`

* `sudo nano ~/.ssh/authorized_keys` and paste key contents

* Give permissions: `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`

## Prepare to deploy project

#### Configure local timezone to UTC
`sudo timedatectl set-timezone UTC`

#### Install & configure Apache to serve Python mod_wsgi app

```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi-py3
```

#### Install & configure PostgreSQL
* Install PostgreSQL: `sudo apt-get install postgresql`

* Switch to `postgres` user: `sudo su - postgres` 

* Open PostgreSQL terminal: `psql`

* Create database user named `catalog` in PostgreSQL with limited db permissions:

```
CREATE ROLE catalog WITH LOGIN PASSWORD 'catalog';
ALTER ROLE catalog CREATEDB;
```

* Exit psql and return to user `grader`

* Create user `catalog`: `sudo adduser catalog`

* Give `catalog` permission to sudo by running `sudo visudo` and adding: `catalog  ALL=(ALL:ALL) ALL`    
in the privileges specification section

* Login as user `catalog`: `su - catalog` and create database: `createdb catalog`

#### Install git
* Login as `grader` and run `sudo apt-get install git`

## Deploy the Item Catalog project

#### Clone and setup Item Catalog project
* Create `catalog` directory and make `grader` the owner:
```
cd /var/www
sudo mkdir catalog
sudo chown -R grader:grader catalog
```

* Clone project:
```
cd catalog
git clone https://github.com/BenGoBlue05/item_catalog catalog
```

#### Setup Virtual Env
```
sudo apt install python3-pip
sudo pip3 install virtualenv 
sudo virtualenv venv
source venv/bin/activate
sudo chmod -R 777 venv
pip3 install flask sqlalchemy google-api-python-client flask-sqlalchemy
```   

#### Create .wsgi file

* Create file: ` sudo nano catalog.wsgi`

* Add the following to the newly created file:
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/catalog")

from app import app as application
application.secret_key = 'super_secret_key'
```

* Enable `wsgi` module: `sudo a2enmod wsgi`

#### Configure virtual host

* Create config file: `sudo nano /etc/apache2/sites-enabled/catalog.conf`

* Add the following to the newly created file:
```
<VirtualHost *:80>
    ServerName 18.222.114.3
    ServerAlias ec2-18-222-114-3.us-east-2.compute.amazonaws.com
    ServerAdmin [EMAIL ADDRESS HERE]
    WSGIDaemonProcess catalog python-path=/var/www/catalog/venv/bin/python
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog/catalog.wsgi
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


* Enable site catalog: `sudo a2ensite catalog`

* Disable default host: `a2dissite 000-default.conf`

#### Edit web app
* Replace db references in files `database_setup.py` and `app.py` from   
`engine = create_engine('sqlite:///catalog.db')`   
to `engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`

*  Configure OAuth client for Google Sign for http://ec2-18-222-114-3.us-east-2.compute.amazonaws.com and add credentials to `client_secrets.json`

## Resources
 * [Integrating Google Sign-In Into your web app](https://developers.google.com/identity/sign-in/web/sign-in#before_you_begin)
 
 * [DigitalOcean - How To Deploy a Flask App on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)















