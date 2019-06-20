# Linux Server Configuration Overview
This project allows you to secure and set up a Linux server and secure it, install and configure a database server, and deploy one of your existing web applications onto it, in this particular one we're going to deploy our Item Catalog project

# Linux Server Instance
To set up the Linux server instance. We recommend using Amazon Lightsail for this. If you don't already have an Amazon Web Services account, set up one by signing up in [Amazon Lightsail](https://signin.aws.amazon.com/signin?redirect_uri=https%3A%2F%2Flightsail.aws.amazon.com%2Fls%2Fwebapp%3Fstate%3DhashArgs%2523%26isauthcode%3Dtrue&client_id=arn%3Aaws%3Aiam%3A%3A015428540659%3Auser%2Fparksidewebapp&forceMobileApp=0) then follow the [documentation](https://lightsail.aws.amazon.com/ls/docs/en_us/articles/getting-started-with-amazon-lightsail) to set up an instance then connect into it using **ssh**

# Network Informations

IP address: **52.28.181.156**
SSH port: **2200**
Application url: **http://www.catalogappstore.com**

# Configuration

### Update all packages
Update all your current packages using:
```
sudo apt-get update && sudo apt-get upgrade
```

### Firewall configuration
Add port **2200** in your **Lightsail** firewall then configure the  uncomplicated firewall **(ufw)** in your instance to only allow incoming connections for **SSH** (port 2200), **HTTP** (port 80), and **NTP** (port 123):
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow 123/udp
sudo ufw enable
```

Change Port **22** to Port **2200** in **sshd_config** , save & quit:
```
sudo nano /etc/ssh/sshd_config
```

### Create a new User 'grader'
Create a new user named **grader**:
```
sudo adduser grader
```

Check **grader** information using **finger tool**: 
```
sudo apt-get install finger
finger grader
```

Give **grader** permission to **sudo** by creating **grader** file in **sudoers.d**
```
sudo nano /etc/sudoers.d/grader
```

then add this line and save the file.
```
grader ALL=(ALL) NOPASSWD:ALL
```

### Create SSH key pair for user 'grader'
Generate SSH keys on your local machine using **ssh-keygen** tool then save the private key in **~/.ssh** and generate a **catchphrase**.
In your server switch session to **grader**, create a **.ssh** directory, generate an **authorized_keys** file then copy the ssh key generated in your local machine into this file:
```
sudo su - grader
mkdir .ssh
nano .ssh/authorized_keys
```

Update file permisions to these files:
```
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```

Reload SSH: 
```
service ssh restart
```

Deactivate login via password by accessing **sshd_config** and changing **passwordAuthentication** from **yes** to **no**
```
sudo nano /etc/ssh/sshd_config
```

When you finish all try to ssh login with the new user **grader** using port **2200** and your ssh key.
```
ssh grader@52.24.125.52 -p 2200 -i [KeyPathOnLocalMachine]
```

KeyPathOnLocalMachine ex: ~/.ssh/key

### Timezone

Configure the local time zone to UTC using:
```
sudo dpkg-reconfigure tzdata
```

### Install Apache and Python mod_wsgi
```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi-py3
```

### Install and configure PostgreSQL
Install PostgreSQL 
```
sudo apt-get install postgresql
```

then Check if no remote connections are allowed:
```
sudo nano /etc/postgresql/10/main/pg_hba.conf
```

Login as user "postgres" then go to postgreSQL shell and create a database and a user both named 'catalog' 
```
sudo su - postgres
psql
postgres=# CREATE DATABASE catalog;
postgres=# CREATE USER catalog;
```

Set a password for user 'catalog' and permissions to "catalog" application database
```
postgres=# ALTER ROLE catalog WITH PASSWORD 'password';
postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
```

Quit postgreSQL and exit
```
postgres=# \q
exit
```
 
### Install git and setup Catalog project.
Install Git using 
```
sudo apt-get install git
```

Move to the **/var/www** directory, create the application directory **catalog** which will contain the **`catalog.wsgi`** and another **catalog** directory for our app content

```
cd /var/www
sudo mkdir catalog
```

Clone your catalog app project in a separate directory using
```
git clone https://github.com/alaared1/fullstack-nanodegree-vm
```

After cloning, move to **catalog** folder and copy its content in **/var/www/catalog/catalog**

##### **Important:** 
**The reason we're cloning the repository in a seperate folder is because its is located inside a vagrant virtual machine and we don't need that**

Move to **/var/www/catalog/catalog** and rename **`application.py`** to **`__init__.py`** using: 
```
sudo mv website.py __init__.py
```

Edit **`database_setup.py`**, **`catalog.py`**, **`run-categories.py`** and **`run-queries.py`** by changing:
```
engine = create_engine('sqlite:///catalogapp.db')
```
to 
```
engine = create_engine('postgresql://catalog:password@localhost/catalog')
```

Install pip3

```
sudo apt-get install pip3`
```

Create database schema
```
sudo python3 database_setup.py
```

Populate the database with some records by executing **`run-categories.py`** then **`run-queries.py`**
```
sudo python3 run-categories.py
sudo python3 run-queries.py
```

## Create the .wsgi File
Create the **.wsgi** File under **`/var/www/catalog`**: 
```
cd /var/www/catalog
sudo nano catalog.wsgi 
```

Add the following lines of code to the **`catalog.wsgi`** file:
	
```
#!/usr/bin/python3
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```

### Configure your Virtual Host
Create **`catalog.conf`** to set the configuration file for the virtual host:
```
sudo nano /etc/apache2/sites-available/catalog.conf
```
Add the following lines of code in that file. 
	
```
<VirtualHost *:80>
    ServerName 52.28.181.156
    ServerAdmin ubuntu@52.28.181.156
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
Enable the virtual host with the following command: 
```
sudo a2ensite catalog
```
Restart Apache 
```
sudo service apache2 restart
```

### Set up a Google Domain
To give your website a domain and to facilitate routing and redirecting, pick a domain using **Google Domains** in this **[link](https://domains.google.com/m/registrar/search)**, proceed with payments and fill your personal info.
On the home page click on **My domains** tab.
Then click on the button **Manage** located in your picked domain row.
Click on **DNS** tab and fill the following data in **Custom resource records** area found at the end of the page:
*   **Name**: www
*   **Type**: A
*   **TTL**: 1d
*   **Data**: [Your Server IP]

Then click **Add**, wait a few seconds then visit your domain address (in my case: www.catalogappstore.com) and you'll visualize your website.

### Configuring Google API Redirecting URLS
Access your [google API Console](https://console.developers.google.com), go to **OAuthConsentScreen** tab and add your domain in **Authorized domains** (ex: catalogappstore.com)
Then go to **Credentials** and add
https://www.catalogappstore.com in **Authorized JavaScript Origins**
https://www.catalogappstore.com/catalog in **Authorised redirect URIs**
Save changes and download the **Cliend ID** json and replace it with the one in the inner **catalog** folder.

### Restart Apache 
```
sudo service apache2 restart
```

YOU'VE MADE IT!

# References

*   https://eu.udacity.com/course/full-stack-web-developer-nanodegree--nd004
*   http://www.islandtechph.com/2017/10/23/how-to-deploy-a-flask-python-3-5-application-on-a-live-ubuntu-16-04-linux-server-running-apache2/
*   https://medium.com/coding-blocks/creating-user-database-and-adding-access-on-postgresql-8bfcd2f4a91e
*   https://docs.sqlalchemy.org/en/13/core/engines.html

# More Information

For more information contact me on [alaa.bouhadjeb@gmail.com](mailto:alaa.bouhadjeb@gmail.com)
