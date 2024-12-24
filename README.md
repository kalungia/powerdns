# PowerDNS Installation on Ubuntu 22.04 LTS
This guide will walk you through the process of installing PowerDNS and configuring it with the MySQL backend on Ubuntu 22.04 LTS. It includes steps for setting up the MariaDB database, configuring PowerDNS, installing necessary dependencies, and setting up the PowerDNS Admin web interface.

### Update Server
```sudo apt update && sudo apt upgrade -y```

## 1. Install and Configure MariaDB
First, we need to install and configure the MariaDB database, which will serve as the backend for PowerDNS.


```bash
sudo apt install mariadb-server -y
Check the status of the MariaDB service:

systemctl status mariadb.service
Secure the MariaDB installation:

sudo mysql_secure_installation
Login to MySQL as root:

sudo mysql -u root -p
```
## 2. Disable systemd-resolved (if required)
You may need to disable systemd-resolved to avoid conflicts with DNS resolution:



sudo systemctl stop systemd-resolved.service
sudo systemctl disable systemd-resolved.service
sudo vim /etc/resolv.conf
Ensure that resolv.conf points to the correct DNS server, if necessary.

3. Add PowerDNS Repository
Add the PowerDNS APT repository to your system:



sudo vim /etc/apt/sources.list.d/pdns.list
Add the repository URL inside the file and then install the repository keyring:



sudo install -d /etc/apt/keyrings
curl https://repo.powerdns.com/FD380FBB-pub.asc | sudo tee /etc/apt/keyrings/auth-49-pub.asc
sudo apt-get update
Install PowerDNS server and MySQL backend:



sudo apt install pdns-server pdns-backend-mysql -y
4. Check PowerDNS Installation
Check the status of the PowerDNS service:



sudo systemctl status pdns.service
5. Configure PowerDNS to Use MySQL Backend
To configure PowerDNS to use the MySQL backend, make sure you modify the PowerDNS configuration files.

Move the default bind.conf file:


mv bind.conf bind.conf.original
Edit the pdns.local.gmysql.conf configuration file:


sudo vim pdns.local.gmysql.conf
Make necessary adjustments in the configuration and then restart PowerDNS:


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
