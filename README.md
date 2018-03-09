# Linux Server Configuration

Linux server configuration project it about installation of a Linux distribution on a virtual machine and prepare it to host your web applications.
This Linux server is an instance on Amazon Lightsail. This project install server updates, secure it and configure web and database server.

* IP address: 18.219.231.12
* SSH port: 2200
* URL: [http://ec2-18-219-231-12.us-east-2.compute.amazonaws.com](http://ec2-18-219-231-12.us-east-2.compute.amazonaws.com)
## SSH into server by following command
* Move the private key file into the folder ~/.ssh (where ~ is your environment's home directory).
```
mv ~/Downloads/LightsailDefaultPrivateKey-us-east-2.pem ~/.ssh/
```
* Open your terminal and type in
```
chmod 600 ~/.ssh/lightsailDefaultPrivateKey-us-east-2.pem
```
```
ssh -i ~/.ssh/LightsailDefaultPrivateKey-us-east-2.pem ubuntu@18.219.231.12
```
## Create user grader
* Add new user
```
$ sudo adduser grader
```
* Create a new file in the sudoers directory with
```
 $ sudo nano /etc/sudoers.d/grader
```
* Add following text
grader ALL=(ALL:ALL) ALL

* Generate key pair locally
```
ssh-keygen
```
* Create new directory
```
 $ su - grader
 $ mkdir .ssh
```
* New file
```
$ touch .ssh/authorized_keys
```
* Run this command on your local machine
```
$ cat .ssh/secureKey.pub
```
copy this key text and paste in authorized_keys file and save it.
```
$ sudo nano .ssh/authorized_keys
```
* Set files permissions
```
$ chmod 700 .ssh
$ chmod 644 .ssh/authorized_keys
```
* Now login as grader user
```
 $ ssh -i ~/.ssh/linuxKey grader@18.219.231.12 -p 2200
```

## Change SSH port
* Update all installed packages
```
$ sudo apt-get update
```
* Install all packages
```
$ sudo apt-get upgrade
```
* Open ssh file to change SSH port
```
$ sudo nano /etc/ssh/sshd_config
```
change Port 22 to 2200
Then run this command
```
$ sudo service ssh restart
```

Warning: When changing the SSH port, make sure that the firewall is open for port 2200 first, so that you don't lock yourself out of the server. Review this video for details! When you change the SSH port, the Lightsail instance will no longer be accessible through the web app 'Connect using SSH' button. The button assumes the default port is being used. There are instructions on the same page for connecting from your terminal to the instance. Connect using those instructions and then follow the rest of the steps.

## Configure the Uncomplicated Firewall(UFW)
* First deny all incoming requests
```
 $ sudo ufw default deny incoming
```
* Allow all outgoing
```
$ sudo ufw default allow outgoing
```
* Check ufw status
```
$ sudo ufw status
```
* Allow incoming TCP packets on port 2200 (SSH):
 ```
 $ sudo ufw allow 2200/tcp
 ```
* Allow incoming TCP packets on port 80 (HTTP):
 ```
 $ sudo ufw allow 80/tcp
 ```
* Allow incoming UDP packets on port 123 (NTP):
```
$ sudo ufw allow 123/udp
```
* Enable firewall
```
 $ sudo ufw enable
```
* Check status
```
$ sudo ufw status
```
* Restart SSH
```
 $ sudo service ssh restart
```
* Disable password based logins
```
$ sudo nano /etc/ssh/sshd_config
```
edit this line of text in file
PasswordAuthentication yes to PasswordAuthentication no

* Restart the service
```
 $ sudo service ssh restart
```
Now all users are enforced to login using a key pair

## Prepare to deploy the project
* Configure the local timezone to UTC
```
$ sudo dpkg-reconfigure tzdata
```
* Install and configure apache2
```
$ sudo apt-get install apache2
```
* install mod_wsgi
Run
```
$ sudo apt-get install libapache2-mod-wsgi python-dev
```
Enable mod_wsgi with
```
$ sudo a2enmod wsgi
```
Start the web server with
```
$ sudo service apache2 start
```
* Install and configure postgresql
```
$ sudo apt-get install postgresql
$ sudo su - postgres
postgres=# psql
postgres=# CREATE DATABASE catalog;
postgres=# CREATE USER catalog;
postgres=# ALTER ROLE catalog WITH PASSWORD 'password';
postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
postgres=# \q
postgres=# exit
```
Postgresql do not allow remote connections by default.
We can double check that no remote connections are allowed by looking in the host based authentication file:
```
$ sudo nano /etc/postgresql/9.1/main/pg_hba.conf
```
[helpful link](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet#code)

* Install Git
```
$ sudo apt-get install git
```
## Deploy the Item Catalog project
```
$ cd /var/www/
$ sudo mkdir catalog
```
* Change owner of the newly created catalog folder
```
$ sudo chown -R grader:grader catalog
```
* Clone your project from github
```
$ git clone [https://github.com/noshi1/catalog.git](https://github.com/noshi1/catalog.git) catalog
```

Then
```
$ cd catalog
```

* Rename application.py to init.py
```
mv application.py __init__.py
```

* Change create engine line in your __init__.py and models.py to:
  engine = create_engine('postgresql://catalog:password@localhost/catalog')
```
sudo nano python /var/www/catalog/catalog/models.py
sudo nano python /var/www/catalog/catalog/__init__.py
```
* Install other project dependencies
```
$ sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils requests
```
* Update path of client_secrets.json file
```
$ sudo nano __init__.py
```

## Create a new catalog.wsgi
```
$ cd /var/www/catalog
$ sudo nano catalog.wsgi
```
* Add the following text
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```
### Configure and Enable a New Virtual Host
* Install virtualenv
```
$ sudo pip install virtualenv
$ sudo virtualenv venv
```
* Change permissions
```
$ sudo chmod -R 777 venv
```
* Activating the virtual environment
```
$ source venv/bin/activate
```
* Install Flask
```
$ sudo apt-get install python-pip
```
* Issue the following command in your terminal:
```
$ sudo nano /etc/apache2/sites-available/catalog
$ sudo nano /etc/apache2/sites-available/catalog.conf
```
Paste this code:
```
<VirtualHost *:80>
    ServerName 18.219.231.12
    ServerAlias http://ec2-18-219-231-12.us-east-2.compute.amazonaws.com
    ServerAdmin admin@18.219.231.12
    WSGIProcessGroup catalog
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
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

Save and close the file.
* Enable the virtual host with the following command:
```
$ sudo a2ensite catalog
```
* Change client_secrets.json path to /var/www/catalog/catalog/client_secrets.json

### Restart Apache
```
$ sudo service apache2 restart
```
#### Visit site at[http://ec2-18-219-231-12.us-east-2.compute.amazonaws.com](http://ec2-18-219-231-12.us-east-2.compute.amazonaws.com)

## Helpful links
* [https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
* [https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04)
