# PowerDNS Installation on Ubuntu 22.04 LTS
This guide will walk you through the process of installing PowerDNS and configuring it with the MySQL backend on Ubuntu 22.04 LTS. It includes steps for setting up the MariaDB database, configuring PowerDNS, installing necessary dependencies, and setting up the PowerDNS Admin web interface.

## Install Update and Upgrade Server
```bash
sudo apt update && sudo apt upgrade -y
```

## Install and Configure MariaDB
First, we need to install and configure the MariaDB database, which will serve as the backend for PowerDNS.

```bash
sudo apt install mariadb-server -y
```
Check the status of the MariaDB service and fix any Errors:
```bash
systemctl status mariadb.service
```

Secure the MariaDB installation:
```bash
sudo mysql_secure_installation
```
When you run this command, the server will show the following prompts. Please follow the steps as shown below to complete the setup correctly.

Enter current password for root: (Enter your SSH root user password)<br>
Switch to unix_socket authentication [Y/n]: Y<br>
Change the root password? [Y/n]: Y<br>
It will ask you to set new MySQL root password at this step. This can be different from the SSH root user password.
Remove anonymous users? [Y/n] Y<br>
Disallow root login remotely? [Y/n]: N<br>
Remove test database and access to it? [Y/n]: Y<br>
Reload privilege tables now? [Y/n]: Y<br>

Login to MySQL as root:
```bash
sudo mysql -u root -p
```
Create Powerdns Database and User then add the Default Schema
```bash
CREATE DATABASE powerdns;
GRANT ALL ON powerdns.* TO 'powerdns'@'localhost' IDENTIFIED BY 'someStrongPassword';
FLUSH PRIVILEGES;
USE powerdns;
```
Default Schema for MYSQL provided by Powerdns [here](https://doc.powerdns.com/authoritative/backends/generic-mysql.html#default-schema)
```bash
CREATE TABLE domains (
  id                    INT AUTO_INCREMENT,
  name                  VARCHAR(255) NOT NULL,
  master                VARCHAR(128) DEFAULT NULL,
  last_check            INT DEFAULT NULL,
  type                  VARCHAR(8) NOT NULL,
  notified_serial       INT UNSIGNED DEFAULT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' DEFAULT NULL,
  options               VARCHAR(64000) DEFAULT NULL,
  catalog               VARCHAR(255) DEFAULT NULL,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE UNIQUE INDEX name_index ON domains(name);
CREATE INDEX catalog_idx ON domains(catalog);


CREATE TABLE records (
  id                    BIGINT AUTO_INCREMENT,
  domain_id             INT DEFAULT NULL,
  name                  VARCHAR(255) DEFAULT NULL,
  type                  VARCHAR(10) DEFAULT NULL,
  content               VARCHAR(64000) DEFAULT NULL,
  ttl                   INT DEFAULT NULL,
  prio                  INT DEFAULT NULL,
  disabled              TINYINT(1) DEFAULT 0,
  ordername             VARCHAR(255) BINARY DEFAULT NULL,
  auth                  TINYINT(1) DEFAULT 1,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE INDEX nametype_index ON records(name,type);
CREATE INDEX domain_id ON records(domain_id);
CREATE INDEX ordername ON records (ordername);


CREATE TABLE supermasters (
  ip                    VARCHAR(64) NOT NULL,
  nameserver            VARCHAR(255) NOT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' NOT NULL,
  PRIMARY KEY (ip, nameserver)
) Engine=InnoDB CHARACTER SET 'latin1';


CREATE TABLE comments (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  name                  VARCHAR(255) NOT NULL,
  type                  VARCHAR(10) NOT NULL,
  modified_at           INT NOT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' DEFAULT NULL,
  comment               TEXT CHARACTER SET 'utf8' NOT NULL,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE INDEX comments_name_type_idx ON comments (name, type);
CREATE INDEX comments_order_idx ON comments (domain_id, modified_at);


CREATE TABLE domainmetadata (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  kind                  VARCHAR(32),
  content               TEXT,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE INDEX domainmetadata_idx ON domainmetadata (domain_id, kind);


CREATE TABLE cryptokeys (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  flags                 INT NOT NULL,
  active                BOOL,
  published             BOOL DEFAULT 1,
  content               TEXT,
  PRIMARY KEY(id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE INDEX domainidindex ON cryptokeys(domain_id);


CREATE TABLE tsigkeys (
  id                    INT AUTO_INCREMENT,
  name                  VARCHAR(255),
  algorithm             VARCHAR(50),
  secret                VARCHAR(255),
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE UNIQUE INDEX namealgoindex ON tsigkeys(name, algorithm);
```
Exit From MySQL
```bash
exit
```

## Disable systemd-resolved (if required)
You may need to disable systemd-resolved to avoid conflicts with DNS resolution:

```bash
sudo systemctl stop systemd-resolved.service
sudo systemctl disable systemd-resolved.service
```

## Ensure that resolv.conf points to the correct DNS server, if necessary.
```bash
sudo vim /etc/resolv.conf
```
Comment Out the nameservers
```
#nameserver 127.0.0.53
#options edns0 trust-ad
#search some domain
```
Replace it with
```
nameserver 8.8.8.8
nameserver 1.1.1.1
```
Update Server after changing nameservers
```bash
sudo apt update
```

## Add PowerDNS Repository
### Add the PowerDNS APT repository to your system:

Create the file '/etc/apt/sources.list.d/pdns.list' with this content:
```bash
deb [signed-by=/etc/apt/keyrings/auth-49-pub.asc] http://repo.powerdns.com/ubuntu jammy-auth-49 main
```
Put this in '/etc/apt/preferences.d/auth-49':
```bash
Package: auth*
Pin: origin repo.powerdns.com
Pin-Priority: 600
```
and execute the following commands:

```bash
sudo install -d /etc/apt/keyrings; curl https://repo.powerdns.com/FD380FBB-pub.asc | sudo tee /etc/apt/keyrings/auth-49-pub.asc &&
sudo apt-get update &&
sudo apt-get install pdns-server pdns-backend-mysql -y
```

## Install Net Tools
```bash
sudo apt install net-tools
```

## Configure PowerDNS to Use MySQL Backend
### To configure PowerDNS to use the MySQL backend, make sure you modify the PowerDNS configuration files.

Backup the default bind.conf file that is under "/etc/powerdns/pds.d":
``cd /etc/powerdns/pds.d``
```bash
mv bind.conf bind.conf.bak
```
Create pdns.local.gmysql.conf
```bash
sudo vim pdns.local.gmysql.conf
```
Add the following:
```bash
# MySQL Configuration
# Launch gmysql backend
launch+=gmysql
# gmysql parameters
gmysql-host=localhost
gmysql-port=3306
gmysql-dbname=powerdns
gmysql-user=powerdns
gmysql-password=userpassword
gmysql-dnssec=yes
# gmysql-socket=
```

sudo systemctl restart pdns.service
sudo systemctl status pdns.service
You can also check for any configuration issues with the following command:



pdns_server --config-dir=/etc/powerdns/ --check-config
6. Set Up PowerDNS Admin Interface (Optional)
If you want to manage PowerDNS via a web interface, you need to install PowerDNS Admin.

Install necessary dependencies:


sudo apt install python3-dev libxml2-dev libxslt1-dev zlib1g-dev libsasl2-dev libldap2-dev build-essential libssl-dev libffi-dev libmysqlclient-dev libjpeg-dev libpq-dev libjpeg8-dev liblcms2-dev libblas-dev libatlas-base-dev
Install Flask and required packages:


curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E  -
pip install -r requirements.txt
Configure the PowerDNS Admin service:


sudo vim /etc/systemd/system/powerdns-admin.service
sudo systemctl daemon-reload
sudo systemctl restart powerdns-admin
sudo systemctl enable powerdns-admin
sudo systemctl status powerdns-admin
Install and configure Nginx:


sudo apt install nginx -y
sudo vim /etc/nginx/conf.d/powerdns-admin.conf
nginx -t
sudo systemctl restart nginx
sudo systemctl enable nginx
sudo systemctl status nginx
7. Testing PowerDNS Installation
To verify that PowerDNS is resolving DNS records, you can run a query with dig:



dig @127.0.0.1 imbraintl.com A
If you see a successful answer, PowerDNS is working as expected.

8. Troubleshooting
If you run into any issues, check the logs:



sudo journalctl -xeu pdns.service
