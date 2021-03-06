
# Linux Web Server Configuration Project
Hello! Welcome to the documentation of the setup of my Linux Web Server for the Item Catalog flask application. It is deployed using an AWS Lightsail instance. Here is the link to the final website: [ItemCatalog](http://13.233.239.144.xip.io)

(Static) IP address for this instance: 13.233.239.144
SSH Port: 2200

Complete url: http://13.233.239.144.xip.io

The following programs were used for the configuration of this web server
* [Apache2](https://httpd.apache.org) HTTP Web server
* [Mod WSGI](https://modwsgi.readthedocs.io/en/develop/)
* [PostgreSQL](https://www.postgresql.org) Database Server

## Setup
Firstly, create an [Amazon web services](https://portal.aws.amazon.com/billing/signup#/start) Lightsail Instance and chose the Ubuntu 16 instance image

Then convert your public ip address into a static public ip address. 

Download the default key and note the full path. Then using the vagrant vm, ssh into your new instance using:

`ssh -i ~/pathtodefaultkey/ username@publicipaddress`

Secondly, upgrade all existing packages
```sh
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install aptitude
sudo aptitude update
sudo aptitude safe-upgrade
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Thirdly, change the ssh port from port 22 to prt 2200. Do this by changing the '/etc/ssh/sshd_config' file and under listening ports, change the port from port 22 to port 2200.

Then run `sudo service sshd restart'

Next, configure the Firewall by running the following lines:
```sh
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw deny ssh
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow ntp
```
And finally `sudo ufw enable`

Click on the Manage option of the Amazon Lightsail Instance, then the Networking tab, and then change the firewall configuration to match the firewall settings above. Allow ports 80(TCP), 123(UDP), and 2200(TCP), and deny the default port 22.

Now you can ssh into the server using
`ssh -i ~/pathtodefaultkey/ username@publicipaddress -p 2200`

### Create new user
Create a new user `grader` using `sudo adduser grader`. When prompted use `grader` as password.

Give grader sudo access using `sudo touch /etc/sudoers.d/grader` and the edit the file using `sudo nano /ect/sudoers.d/grader` and add the following line 
```sh
grader ALL=(ALL) NOPASSWD:ALL
```
Now, make a public key for grader
- On the local machine:
  * Use ssh-keygen to make a pair of public and private key for the grader user
  * Paste the private key in the local machines .ssh folder
  * Copy the contents of the public key and log into your grader account
- On the grader vm:
  * Make a new dir `~/.ssh`
  * Run `sudo nano ~/.ssh/authorized_keys` and paste the contents of the public key into file and save
  * Change the permissions: `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
  * Check in `/etc/ssh/sshd_config` file if PasswordAuthentication is set to no
  * Restart ssh with `sudo service ssh restart`
- On the local machine log in with :
```sh
ssh -i ~.ssh/private_key grader@ipaddress -p 2200
```
and use passphrase 'grader'

### Configure TimeZone

Run the following command to change the time zone to UTC: 
```sh
sudo dpkg-reconfigure tzdata
```
Then, in the 'other' options, select UTC

### Install Git
Use the following command to install git 
`sudo apt-get install git`

Clone repository into the dir /var/www
```sh
cd /var/www/
sudo git clone https://github.com/FizzaKAC/Item-Catalog.git
```
### Install Apache
Install apache using the following command: 
```sh
sudo apt-get install apache2
```

Then, install mod-wsgi:
```sh
sudo apt-get install libapache2-mod-wsgi python-dev
```

Make an `itemscatalog.wsgi` file in the application dir. Paste the following code:
```sh
import sys
sys.path.insert(0, "/var/www/Item-Catalog")
from views import app as application
application.secret_key='super_secret_key'
```

 Set up a virtual host file: `cd /etc/apache2/sites-available/catalogapp.conf`
 Then paste the following code in it:
```sh
<VirtualHost *>
 ServerName example.com
 WSGIScriptAlias / /var/www/Item-Catalog/itemcatalog.wsgi
 WSGIDaemonProcess hello
 <Directory /var/www/Item-Catalog>
  WSGIProcessGroup hello
  WSGIApplicationGroup %{GLOBAL}
   Order deny,allow
   Allow from all
 </Directory>
</VirtualHost>
```
### Install Flask
```sh
sudo pip install requests 
sudo pip install oauth2client 
sudo pip install httplib2
sudo apt-get install python-pip
sudo apt-get python-sqlalchemy 
sudo apt-get python-flask 
sudo apt-get python-psycopg2
```

### Install PostgreSQL
Install postgreSQL using the following command `sudo apt-get install postgresql`. Make sure remote connections arent possible by checking the `sudo nano /etc/postgresql/9.5/main/pg_hba.conf` config file. It should have the following code:
```sh
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```
#### Create database
Log into the postgres user. First change the default password using `sudo passwd postgres` and enter postgres as password. Then log in using sudo postgres and enter the password.

Once logged in, enter psql to enter the postgresql environment. Run the following commands:
```sh
CREATE USER catalog WITH PASSWORD 'catalog';
CREATE DATABASE catalogapp;
GRANT ALL PRIVILEGES ON DATABASE catalogapp TO catalog;
```
Then, `exit` the postgres user and update the database model and all other files to reflect the changes on out engine:
```sh
engine = create_engine('postgresql://catalog:catalog@localhost/catalogapp')
```
Now run the `models.py` file to create the database tables and `lotsOfCategories.py` to populate the categories.

### OAuth
To set up oauth, we will be using xip.io to create wildcard dns for our static ip address. We will first authorize out xip.io domain on the google developers console website, then add the authorized javscript origins and redirect uris. Next, we will replace the client_secrets file with the new client secrets file, and update the client if in our `login.html` file respectivly.

We also need to show the full path of the client_secrets.json file in our main python file :
```sh
CLIENT_ID = json.loads(
    open('/var/www/item-Catalog/client_secrets.json', 'r').read())['web']['client_id']

oauth_flow = flow_from_clientsecrets('/var/www/item-Catalog/client_secrets.json', scope='')
```

### FAIL2BAN
Added after first review of project. [Fail2Ban](https://www.fail2ban.org/wiki/index.php/Main_Page) scans log files and bans IPs that show the malicious signs -- too many password failures, seeking for exploits, etc. Generally Fail2Ban is then used to update firewall rules to reject the IP addresses for a specified amount of time. 

Install Fail2Ban:
```sh 
sudo apt-get install fail2ban
```
Install sendmail and iptables:
```sh
sudo apt-get install sendmail iptables-persistent
```
Create a copy of the original config file and edit some default vaules:
```sh
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
set bantime = 600
destemail = xyz@domain.com
action = %(action_mwl)s 
```
Under `[sshd]` change `port = ssh` to `port = 2200`

Restart the service: 
```sh
sudo service fail2ban restart
```
You will then recieve an email saying that fail2ban has been started(You might need to check your spam)

## Resources
- [Amazon Lightsail](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/4/html/Security_Guide/s2-wstation-privileges-noroot.html)
- [Apache Web Server](https://httpd.apache.org)
- [PostgreSql Web Server](postgresql.org)
- [VirtualBox and Vagrant](https://www.vagrantup.com/docs/virtualbox/)
- [Fail2Ban](https://www.fail2ban.org/wiki/index.php/Main_Page)
- [Changing SSH Port](https://pk.godaddy.com/help/changing-the-ssh-port-for-your-linux-server-7306)
- [Adding privilages to DB user](https://www.ntchosting.com/encyclopedia/databases/postgresql/create-user/)
- [Xip.io](http://xip.io)
- [Setting up DB](https://www.techrepublic.com/blog/diy-it-guy/diy-a-postgresql-database-server-setup-anyone-can-handle/)
- [Creating Readme on dillinger](https://dillinger.io)
- [How to secure PostgreSQL](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
- [Disallowing root access on our server](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/4/html/Security_Guide/s2-wstation-privileges-noroot.html)
- [How to install Fail2Ban](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)
- [Udacity writing READMES free course](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/4/html/Security_Guide/s2-wstation-privileges-noroot.html)
