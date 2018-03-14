# Application Server Setup Notes
*Notes for setting up a Linux Server (Ubuntu 16.04 LTS) to run a **N-Tier Web Application** with the following components:*
* **MongoDB** (3.x) NoSQL Document Database Server (Port 27017)
* **uWSGI** Python Application Server (to Host Django REST API)
* **NGINX** Reverse Proxy and Web Server (To serve both the API and Client App [Angular] over a single http port [80])

**This server configuration is using Python 3**

## MongoDB Database Server Installation & Configuration
**MongoDB**, a widely-used Open Source NoSQL database system, is used to store the Box Loading status information.  The REST API will process incoming **CRUD** (**C**reate **R**ead **U**pdate **D**elete) requests (mapped from http verbs: POST, GET, PUT, & DELETE) using the **PyMongo** library for MongoDB (https://api.mongodb.com/python/current/). The PyMongo library allows the Django Python code to connect and submit quereies to the MongoDB database.  The bl-status system uses the freely available **Community Edition** of MongoDB.  The current version is: **[3.4]**. 

The first step is to import the MongoDB public GPG/PGP key file into the Ubuntu Package Manager (to verify the authentcity of the software package):
```
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 0C49F3730359A14518585931BC711F9BA15703C6
```
The output from this command should be similar to what is shown below:

```
Executing: /tmp/tmp.Q8XWsuPMns/gpg.1.sh --keyserver
hkp://keyserver.ubuntu.com:80
--recv
0C49F3730359A14518585931BC711F9BA15703C6
gpg: requesting key A15703C6 from hkp server keyserver.ubuntu.com
gpg: key A15703C6: public key "MongoDB 3.4 Release Signing Key <packaging@mongodb.com>" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
```

Next, create a new Package Manager repository list file for MongoDB:
```
$ echo "deb [ arch=amd64,arm64 ] http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.4.list
```
Run the Package Manger's *update* process to force the inclusion of the newly-added MongoDB repository:
```
$ sudo apt-get update
```
Next, download/install the MongoDB software:
```
$ sudo apt-get install -y mongodb-org
```
After installing the MongoDB, it needs to be configured to automatically start up when the Operating System boots (or re-boots). In order to do this, the Ubuntu Linux service manager **(systemd)** must be confifured to manage MongoDB (taken from:	https://docs.mongodb.com/manual/tutorial/install-mongodb-enterprise-on-ubuntu/ & https://www.mkyong.com/mongodb/mongodb-allow-remote-access/):

- A new systemd service configuration file must be created by issuing the command below:
```
$ sudo nano /etc/systemd/system/mongodb.service
```
- Paste the following contents into the text file:
```
[Unit]
Description=High-performance, schema-free document-oriented database
After=network.target

[Service]
User=mongodb
ExecStart=/usr/bin/mongod --quiet --config /etc/mongod.conf

[Install]
WantedBy=multi-user.target
```
- Next, start the newly-defined service:
```
$ sudo systemctl start mongodb
```
- to verify the service running, type the following command:
```
$ sudo systemctl status mongodb
```
The output should be similar to below:
```
mongodb.service - High-performance, schema-free document-oriented database
   Loaded: loaded (/etc/systemd/system/mongodb.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2016-04-25 14:57:20 EDT; 1min 30s ago
 Main PID: 4093 (mongod)
    Tasks: 16 (limit: 512)
   Memory: 47.1M
      CPU: 1.224s
   CGroup: /system.slice/mongodb.service
           └─4093 /usr/bin/mongod --quiet --config /etc/mongod.conf
```
- Finally, enable the service to automatically start when the server boots:
```
$ sudo systemctl enable mongodb
```

The output should be as shown below:
```
Created symlink from /etc/systemd/system/multi-user.target.wants/mongodb.service to /etc/systemd/system/mongodb.service.
```

For management, testing, and diagnostic purposes, external/LAN access to the MongoDB server is required.  The MongoDB IP binding configuration setting must be updated to allow outside/LAN IP access.  Also, the Linux Firewall must be updated to allow external/LAN access to the TCP Port that MongoDB listens on.

To allow external/LAN access to the database: 

- Update the MongoDB configuration file:
```
$ sudo nano /etc/mongod.conf
```
- Change the **bind_ip** parameter to include the **Server's** Static IP address:
```
# Listen to local and LAN interfaces.
bind_ip = 127.0.0.1,172.16.168.110
```
To allow MongoDB traffic through the Linux Firewall, enter the following **ufw** command:
```
$ sudo ufw allow from any to any port 27017
```

## uWSGI Installation and configuration
**uWSGI** is an implimetation of the **Web Server Gateway Interface (WSGI)** Specification. It provides a universial interface 
between a given Web Server (*NGINX* for this setup) and a given Python-based Web Application (*Django REST Framework* 
for this setup).

### Install and Configure VirtualEnv and VirtualEnvWrapper
An isolated runtime environment will need be establised on the server for hosting the Python-based Django REST API Application.  This accomplished using the **VirtualEnv** tool with the **VirtualEnvWrapper** add-on.  The Virtual Environment allows the Application's depencies (Libraries, Frameworks, etc.) to be managed independently of any other Applications that may be provisioned on the same server instance. For example, two Web Applications can run side-by-side with different versions of the Django Framework. To install and configure the tool, via the command-line:

**1. Install the pip Python Package Manager:**
```
$ sudo apt-get update
$ sudo apt-get install python3-pip
```

**2. Upgrade pip to latest version and install VirtualEnv and VirtualEnvWrapper packages:**
```
$ sudo -H pip3 install --upgrade pip
$ sudo -H pip3 install virtualenv virtualenvwrapper
```

**3. Configure the Linux Shell to work with the VirtualEnvWrapper add-on tool:**
```
$ echo "export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3" >> ~/.bashrc
$ echo "export WORKON_HOME=~/Env" >> ~/.bashrc
$ echo "source /usr/local/bin/virtualenvwrapper.sh" >> ~/.bashrc
$ source ~/.bashrc
```
### Install uWSGI
uWSGI will be run in "Emperor mode", which allows a master process to manage separate applications automatically given a set of configuration files.

**1. Install Python Development Files:**
```
$ sudo apt-get install python3-dev
```
**2. Install uWSGI Package**
```
$ sudo -H pip3 install uwsgi
```
**3. Create folder for Site configuration files**
```
$ sudo mkdir -p /etc/uwsgi/sites
```
**4. Create uWSGI logging folder and configure permissions
```
$ sudo mkdir /var/log/uwsgi
$ sudo chown -R netadmin /var/log/uwsgi
```
**uWSGI Site Configuration File Example *(/etc/uwsgi/sites folder):***
```
$ sudo nano /etc/uwsgi/sites/bl-status-api.ini
```

In this configuration: *chdir* points to the project folder. *home* specifies the Virtual Environment folder for the applciation. *module* specifies the the Django uWSGI module file for the application (this file is generated as part of the Django project initialization [*manage.py startproject*]).  uWSGI is set with a *master* service, which means the uWSGI server can be gracefully restarted without closing the main sockets. This functionality allows you patch/upgrade the uWSGI server without closing the connection with the web server and losing a single request. The *processes* parameter specifies the number of instances of the application to be spawned in RAM to handle multiple requests concurrently.  These worker processes are monitored and managed by the *master* process. The Unix *socket* communuications file and it's permissions are specified.  *vacuum* tells uWSGI to "clean up" and release all used resources when shutting down. The uwsgi logging folder is also specified.
```
[uwsgi]
# project folder location
chdir = /home/netadmin/bl-status-api/bl_status_api

# virtual environment (dependencies) folder location
home = /home/netadmin/Env/bl-status-api

# wsgi application definition file name
module = bl_status_api.wsgi:application

# enable master process
master = true

# number of worker processes
processes = 5

# unix socket communications (with Nginx) file
socket = /run/uwsgi/bl_status_api.sock
chown-socket = netadmin:www-data
chmod-socket = 660

# auto clean-up on shutdown
vacuum = true

# logging
logto = /var/log/uwsgi/%n.log
```

### Resgister uWSGI as a system service (systemd)

**create a systemd Unit file:**
```
$ sudo nano /etc/systemd/system/uwsgi.service
```

**uWSGI sytemd Unit Configuration File Example:**
This example describes the service and specifies the location of the socket communication data ("/run/uwsgi") and where the Site definitions are located (for management). It's tells systemd to auto-restart the service when a critial failure occurs.  The service will run at "normal" Linux run levels (2, 3, & 4):
```
[Unit]
Description=uWSGI Emperor service

[Service]
# set up Unix Socket communications
ExecStartPre=/bin/bash -c 'mkdir -p /run/uwsgi; chown netadmin:www-data /run/uwsgi'
# enable emperor mode: location for sites to manage
ExecStart=/usr/local/bin/uwsgi --emperor /etc/uwsgi/sites
# auto-restart
Restart=always
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all

[Install]
# available at Linux runlevels 2, 3, & 4
WantedBy=multi-user.target
```

## Install and Configure Nginx as a Reverse Proxy
The **Nginx** Web Server and Reverse Proxy sits in front the the uWSGI Application Server *(exposed to the public interface)*. Incoming requests and outgoing responses *(over http Port 80 in this case)* are routed by Nginx to and from the uWSGI Application Server via a **Unix socket** connection.  This socket-based internal communication has certain performance and security advantages over using http *(a http option is avaible, if needed)*.  As a *Reverse Proxy*, Nginx allows mulitple sites to be hosted on a single TCP Port *(80 in this case)*. It does this by examining the destination url in the incoming http request *(e.g. "bl-status-api.emsmail.com")* and routing the request to site that is configured to respond to that URL. 

**1. Install Nginx:**
```
$ sudo apt-get install nginx
```

**2. Create Nginx site configuration file for the Django/uWSGI REST API (bl-status-api):**
```
$ sudo nano /etc/nginx/sites-available/bl-status-api
```

**Nginx Django/uWSGI Site Configuration File Example:**
The site listens on port 80 (http) for requests directed at the URL for the API. Logging data is saved in the Linux Home folder for the autoritative User account (*these folders need to be already created prior to starting the Nginx service*).  An error will not be logged for Sites without an icon image file (*favicon.ico*).  The Django folder containing the application's static assets is defined, along with the root entry folder for the application. The uWSGI Unix socket communication parameters are also specified:
```
server {
    # listen for connections directed at the API URL (Django app)
    listen 80;
    server_name bl-status-api.emsmail.com;

    # specify location for log files
    access_log /home/netadmin/bl-status-logs/api/nginx_access.log;
    error_log /home/netadmin/bl-status-logs/api/nginx_error.log;

    # ignore/bypass application icon (favicon) not-found error
    location = /favicon.ico { access_log off; log_not_found off; }

    # specify location for Django static files
    location /static/ {
        root /home/netadmin/bl-status-api/bl_status_api;
    }

    # specify Unix Socket for Nginx/Django comunications
    location / {
        include         uwsgi_params;
        uwsgi_pass      unix:/run/uwsgi/bl_status_api.sock;
    }
}
```

**3. Create Nginx site configuration file for the Angular App (bl-status-app):**
```
$ sudo nano /etc/nginx/sites-available/bl-status-app
```

**Nginx Angular Web Client App Configuration File Example:**
The site listens on port 80 (http) for requests directed at the URL for the Angular Client-side Appliction. Logging data is saved in the Linux Home folder for the autoritative User account (*these folders need to be already created prior to starting the Nginx service*).  An error will not be logged for Sites without an icon image file (favicon.ico). The location is specified for the root folder of the static Angular Web App files (HTML, JavaScript, CSS):
```
server {
    # listen for connections directed at the Client App URL (Angular)
    listen 80;
    server_name bl-status.emsmail.com;

    # specify location for log files
    access_log /home/netadmin/bl-status-logs/app/nginx_access.log;
    error_log /home/netadmin/bl-status-logs/app/nginx_error.log;

    # ignore/bypass application icon (favicon) not-found error
    location = /favicon.ico { access_log off; log_not_found off; }

    # location of the Angular Client Application
    location / {
        root /home/netadmin/bl-status-app/;
        index index.html;
    }
}
```

**4. Enable the Sites:**
Upon starting, the Nginx service, by default, scans a specific folder for the Site configuration file(s) (*/etc/nginx/sites-enabled*).  Nginx then loads all of the available configuration(s). Sites are enabled by copying or linking the given Site configuration file (from */etc/nginx/sites-available*) to the enabled Site folder.  The most common practice is to link (Linux symbolic link)... This is done with the following commands:
```
$ sudo ln -s /etc/nginx/sites-available/bl-status-api /etc/nginx/sites-enabled
$ sudo ln -s /etc/nginx/sites-available/bl-status-app /etc/nginx/sites-enabled
```

To verify that the link worked, type the following command:
```
netadmin@ubuntu:~$ ls -la /etc/nginx/sites-enabled
total 8
drwxr-xr-x 2 root root 4096 Feb 26 12:12 .
drwxr-xr-x 6 root root 4096 Feb 24 07:56 ..
lrwxrwxrwx 1 root root   40 Feb 26 11:55 bl-status-api -> /etc/nginx/sites-available/bl-status-api
lrwxrwxrwx 1 root root   40 Feb 26 12:12 bl-status-app -> /etc/nginx/sites-available/bl-status-app
lrwxrwxrwx 1 root root   34 Feb 24 07:56 default -> /etc/nginx/sites-available/default
netadmin@ubuntu:~$
```
The Site configuration files in the "sites-enabled" folder are are now shown as links (->) to files in the "site-available" folder. The Nginx service will then load the linked configurations automatically at start-up.

## Allow Web traffic, via Nginx, through the Linux Firewall
```
$ sudo ufw allow 'Nginx Full'
```

## Enable uWSGI and Nginx to auto-start on system boot
Now that the services have an initial configfuration, they can be configured to automatically start when the Operating System is started or re-booted.  The following commands achieve this:
```
$ sudo systemctl enable nginx
$ sudo systemctl enable uwsgi
```

To verify the services are enabled, reboot the server (*$ sudo reboot now*), then query the status of the services.  The output should be similar to what is shown below:
```
netadmin@ubuntu:/etc/uwsgi/sites$ sudo systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-03-13 10:15:20 PDT; 1h 11min ago
  Process: 1032 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
  Process: 1022 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
 Main PID: 1035 (nginx)
   CGroup: /system.slice/nginx.service
           ├─1035 nginx: master process /usr/sbin/nginx -g daemon on; master_process on
           ├─1036 nginx: worker process
           └─1037 nginx: worker process

Mar 13 10:15:20 ubuntu systemd[1]: Starting A high performance web server and a reverse proxy server...
Mar 13 10:15:20 ubuntu systemd[1]: Started A high performance web server and a reverse proxy server.


netadmin@ubuntu:/etc/uwsgi/sites$ sudo systemctl status uwsgi
● uwsgi.service - uWSGI Emperor service
   Loaded: loaded (/etc/systemd/system/uwsgi.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-03-13 10:32:35 PDT; 54min ago
  Process: 1261 ExecStartPre=/bin/bash -c mkdir -p /run/uwsgi; chown netadmin:www-data /run/uwsgi (code=exited, status=0/SUCCESS)
 Main PID: 1267 (uwsgi)
   Status: "The Emperor is governing 1 vassals"
   CGroup: /system.slice/uwsgi.service
           ├─1267 /usr/local/bin/uwsgi --emperor /etc/uwsgi/sites
           ├─1270 /usr/local/bin/uwsgi --ini bl-status-api.ini
           ├─1272 /usr/local/bin/uwsgi --ini bl-status-api.ini
           ├─1273 /usr/local/bin/uwsgi --ini bl-status-api.ini
           ├─1274 /usr/local/bin/uwsgi --ini bl-status-api.ini
           ├─1275 /usr/local/bin/uwsgi --ini bl-status-api.ini
           └─1276 /usr/local/bin/uwsgi --ini bl-status-api.ini

Mar 13 10:32:35 ubuntu uwsgi[1267]: your memory page size is 4096 bytes
Mar 13 10:32:35 ubuntu uwsgi[1267]: detected max file descriptor number: 1024
Mar 13 10:32:35 ubuntu uwsgi[1267]: *** starting uWSGI Emperor ***
Mar 13 10:32:35 ubuntu systemd[1]: Started uWSGI Emperor service.
Mar 13 10:32:35 ubuntu uwsgi[1267]: *** has_emperor mode detected (fd: 7) ***
Mar 13 10:32:35 ubuntu uwsgi[1267]: [uWSGI] getting INI configuration from bl-status-api.ini
Mar 13 10:32:35 ubuntu uwsgi[1267]: Tue Mar 13 10:32:35 2018 - [emperor] vassal bl-status-api.ini has been spawned
Mar 13 10:32:35 ubuntu uwsgi[1267]: Tue Mar 13 10:32:35 2018 - [emperor] vassal bl-status-api.ini is ready to accept requests
Mar 13 11:05:03 ubuntu uwsgi[1267]: Tue Mar 13 11:05:03 2018 - [emperor] vassal bl-status-api.ini is now loyal
Mar 13 11:05:32 ubuntu uwsgi[1267]: Tue Mar 13 11:05:32 2018 - [emperor] vassal bl-status-api.ini is now loyal
netadmin@ubuntu:/etc/uwsgi/sites$
```

