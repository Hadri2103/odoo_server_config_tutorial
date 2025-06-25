# Setting up a Odoo 18 ecommerce on a DigitalOcean droplet
Video tutorial linked to this note : LINK

## Create a DigitalOcean droplet
Setup parameters :
- Ubuntu 22.04 (LTS) x64
- Basic plan
- Regular SSD
- 6$/month offer

## Server setup and installing Odoo 18
Launch the server console
Update the server
```
sudo apt-get update
```
Install python and all needed libraries
```
sudo apt-get install -y python3-pip
sudo apt-get install -y python3-dev libxml2-dev libxslt1-dev zlib1g-dev libsasl2-dev libldap2-dev build-essential libssl-dev libffi-dev libmysqlclient-dev libjpeg-dev libpq-dev libjpeg8-dev liblcms2-dev libblas-dev libatlas-base-dev
sudo apt-get install -y npm
sudo ln -s /usr/bin/nodejs /usr/bin/node
sudo npm install -g less less-plugin-clean-css
sudo apt-get install -y node-less
sudo reboot
```
Setup the database
```
sudo apt-get install -y postgresql
sudo su - postgres
createuser --createdb --username postgres --no-createrole --superuser --pwprompt odoo18

Set database password as odoo18

exit
```
System user for Odoo 18
```
sudo adduser --system --home=/opt/odoo18 --group odoo18
```
Get Odoo 18 from git
```
sudo apt-get install -y git
sudo su - odoo18 -s /bin/bash
git clone https://www.github.com/odoo/odoo --depth 1 --branch 18.0 --single-branch .
exit
```
Install required python packages
```
sudo apt install -y python3-venv
sudo python3 -m venv /opt/odoo18/venv
sudo -s
cd /opt/odoo18/
source venv/bin/activate
sudo apt-get install python3-dev build-essential libevent-dev libssl-dev
pip install --upgrade pip setuptools wheel
nano requirements.txt

Modify gevent and greelet requiremnts from > 3.10 to >= 3.10

pip install -r requirements.txt
pip install phonenumbers
sudo wget https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.5/wkhtmltox_0.12.5-1.bionic_amd64.deb
sudo wget http://archive.ubuntu.com/ubuntu/pool/main/o/openssl/libssl1.1_1.1.1f-1ubuntu2_amd64.deb
sudo dpkg -i libssl1.1_1.1.1f-1ubuntu2_amd64.deb
sudo apt-get install -y xfonts-75dpi
sudo apt-get install -f xfonts-base xfonts-75dpi
sudo dpkg -i wkhtmltox_0.12.5-1.bionic_amd64.deb
sudo apt install -f
deactivate
```
Setup the configuration file
```
sudo cp /opt/odoo18/debian/odoo.conf /etc/odoo18.conf
sudo nano /etc/odoo18.conf
```
Copy/paste
```
[options]
; This is the password that allows database operations:
admin_passwd = admin
db_host = localhost
db_port = 5432
db_user = odoo18
db_password = odoo18
addons_path = /opt/odoo18/addons
default_productivity_apps = True
logfile = /var/log/odoo/odoo18.log
```
Give needed permissions 
```
sudo chown odoo18: /etc/odoo18.conf
sudo chmod 640 /etc/odoo18.conf
sudo mkdir /var/log/odoo
sudo chown odoo18:root /var/log/odoo
```
Setup service file
```
sudo nano /etc/systemd/system/odoo18.service
```
Copy/paste
```
[Unit]
Description=Odoo18
Documentation=http://www.odoo.com
[Service]
# Ubuntu/Debian convention:
Type=simple
User=odoo18
ExecStart=/opt/odoo18/venv/bin/python3.10 /opt/odoo18/odoo-bin -c /etc/odoo18.conf
[Install]
WantedBy=default.target
```
Give needed permissions
```
sudo chmod 755 /etc/systemd/system/odoo18.service
sudo chown root: /etc/systemd/system/odoo18.service
```

> Droplet Snaphot Name : *250625-Odoo18_ServerSetup*

You now have a working version of Odoo 18 that you can access at ```http://<your_IP_address>:8069```
Print log file in real-time : ```sudo tail -f /var/log/odoo/odoo18.log```
Start/stop/restart Odoo :
```
sudo systemctl start odoo18.service
sudo systemctl enable odoo18.service
sudo systemctl restart odoo18.service
sudo systemctl stop odoo18.service
```
Continue now with NGINX setup or database creation.

## NGINX
Install NGINX
```
sudo apt install nginx -y
cd /etc/nginx/sites-available/
rm -rf default
sudo nano odoo18.conf
```
Create NGINX config file. Copy/paste.
```
#odoo server
upstream odoo {
server 127.0.0.1:8069;
}
upstream odoochat {
server 127.0.0.1:8072;
}

server {
listen 80;
server_name yourip; #DOMAIN NAME OR IP ADDRESS WITHOUT HTTP

proxy_read_timeout 720s;
proxy_connect_timeout 720s;
proxy_send_timeout 720s;


# Add Headers for odoo proxy mode
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Real-IP $remote_addr;

# log
access_log /var/log/nginx/access.log;
error_log /var/log/nginx/error.log;

# Redirect requests to odoo backend server
location / {
proxy_redirect off;
proxy_pass http://odoo;
}
location /longpolling {
proxy_pass http://odoochat;
}

# common gzip
gzip_types text/css text/less text/plain text/xml application/xml application/json application/javascript;
gzip on;

client_body_in_file_only clean;
client_body_buffer_size 32K;
client_max_body_size 500M;
sendfile on;
send_timeout 600s;
keepalive_timeout 300;
}
```
Add to enabled sites
```
sudo ln -s /etc/nginx/sites-available/odoo18.conf /etc/nginx/sites-enabled/odoo18.conf
cd ..
cd sites-enabled/
rm -rf default
sudo nginx -t
```
Tell Odoo you have NGINX enabled and enable proxy mode
```
sudo nano /etc/odoo18.conf
```
Add at the end of the file
```
proxy_mode=True
```

> Droplet Snaphot Name : TODO

Restart Odoo and NGINX and test your ip
```
cd
sudo service nginx stop
sudo service nginx start
sudo systemctl restart odoo18.service
```

You now have NGINX setup. You can access your working version of Odoo at ```http://<your_IP_address>```

See nginx log in real-time : ```sudo tail -f /var/log/nginx/access.log```

Continue now with domain name setup or database creation.



## Database creation
Connect to your Odoo server (```http://<your_IP_address>:8069``` or ```http://<your_IP_address>``` or ```http://<your_domain>```)
Setup the following informations :
| Field | Description |
| -------- | ------- |
| Master Password | Database master password, choose carefully and write it down |
| Database Name | Something like 'projectname_db' works well |
| Email | Your email for future connexions |
| Password | Your password for future connexions |
| Phone Number | Your phone number |
| Language | Better use english for setup process |
| Country | The country in which the Odoo instance will operate |

Then, connect to the Odoo instance.









