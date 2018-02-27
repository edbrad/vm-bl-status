# Application Server Setup Notes
*Notes for setting up a Linux Server (Ubuntu 16.04 LTS) to run a **N-Tier Web Application** with the following components:*
* **MongoDB** (3.x) NoSQL Document Database Server (Port 27017)
* **uWSGI** Python Application Server (to Host Django REST API)
* **NGINX** Reverse Proxy and Web Server (To serve both the API and Client App [Angular] over a single http port [80])

**This server configuration is using Python 3**

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
**uWSGI Site Configuration File Example *(/etc/uwsgi/sites folder):***
```
$ sudo nano /etc/uwsgi/sites/bl-status-api.ini
```

The configuration uses variables (%) to make the file easy to update and use as a template for setting up other Sites.  The two fields that drive the configuration are the following: 

* **project** *(Django site name)* 
* **uid** *(Linux User ID that has the authority to manage the Application files and services)*

In this configuration: *chdir* points to the project folder. *home* specifies the Virtual Environment folder for the applciation. *module* specifies the the Django uWSGI module file for the application (this file is generated as part of the Django project initialization [*manage.py startproject*]).  uWSGI is set with a *master* service, which means the uWSGI server can be gracefully restarted without closing the main sockets. This functionality allows you patch/upgrade the uWSGI server without closing the connection with the web server and losing a single request. The *processes* parameter specifies the number of instances of the application to be spawned in RAM to handle multiple requests concurrently.  These worker processes are monitored and managed by the *master* process. The Unix *socket* communuications file and it's permissions are specified.  *vacuum* tells uWSGI to "clean up" and release all used resources when shutting down.
```
[uwsgi]
# project/app name
project = bl-status-api
# authoritative user ID
uid = netadmin
# parent folder of project folder
base = /home/%(uid)

# project folder location
chdir = %(base)/%(project)
# virtual environment (dependencies) folder location
home = %(base)/Env/%(project)
# wsgi application definition file name
module = %(project).wsgi:application

# enable master process
master = true
# number of worker processes
processes = 5

# unix socket communications (with Nginx) file
socket = /run/uwsgi/%(project).sock
chown-socket = %(uid):www-data
chmod-socket = 660

# auto clean-up on shutdown
vacuum = true
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
        root /home/netadmin/bl-status-api;
    }

    # specify Unix Socket for Nginx/Django comunications
    location / {
        include         uwsgi_params;
        uwsgi_pass      unix:/run/uwsgi/bl-status-api.sock;
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
        root   /home/netadmin/bl-status-app/;
        index  index.html;
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
netadmin@ubuntu:~$ sudo systemctl status nginx
[sudo] password for netadmin:
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-02-27 11:14:07 PST; 2h 54min ago
  Process: 1029 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
  Process: 1012 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
 Main PID: 1039 (nginx)
   CGroup: /system.slice/nginx.service
           ├─1039 nginx: master process /usr/sbin/nginx -g daemon on; master_process on
           ├─1040 nginx: worker process
           └─1041 nginx: worker process

Feb 27 11:14:07 ubuntu systemd[1]: Starting A high performance web server and a reverse proxy server...
Feb 27 11:14:07 ubuntu systemd[1]: Started A high performance web server and a reverse proxy server.

● uwsgi.service - uWSGI Emperor service
   Loaded: loaded (/etc/systemd/system/uwsgi.service; disabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-02-27 13:25:07 PST; 44min ago
  Process: 1713 ExecStartPre=/bin/bash -c mkdir -p /run/uwsgi; chown netadmin:www-data /run/uwsgi (code=exited, status=0/SUCCESS)
 Main PID: 1717 (uwsgi)
   Status: "The Emperor is governing 1 vassals"
   CGroup: /system.slice/uwsgi.service
           ├─1717 /usr/local/bin/uwsgi --emperor /etc/uwsgi/sites
           ├─1720 /usr/local/bin/uwsgi --ini bl-status-api.ini
           ├─1722 /usr/local/bin/uwsgi --ini bl-status-api.ini
           ├─1723 /usr/local/bin/uwsgi --ini bl-status-api.ini
           ├─1724 /usr/local/bin/uwsgi --ini bl-status-api.ini
           ├─1725 /usr/local/bin/uwsgi --ini bl-status-api.ini
           └─1726 /usr/local/bin/uwsgi --ini bl-status-api.ini

Feb 27 13:25:07 ubuntu uwsgi[1717]: dropping root privileges after application loading
Feb 27 13:25:07 ubuntu uwsgi[1717]: *** uWSGI is running in multiple interpreter mode ***
Feb 27 13:25:07 ubuntu uwsgi[1717]: spawned uWSGI master process (pid: 1720)
Feb 27 13:25:07 ubuntu uwsgi[1717]: spawned uWSGI worker 1 (pid: 1722, cores: 1)
Feb 27 13:25:07 ubuntu uwsgi[1717]: spawned uWSGI worker 2 (pid: 1723, cores: 1)
Feb 27 13:25:07 ubuntu uwsgi[1717]: spawned uWSGI worker 3 (pid: 1724, cores: 1)
Feb 27 13:25:07 ubuntu uwsgi[1717]: spawned uWSGI worker 4 (pid: 1725, cores: 1)
Feb 27 13:25:07 ubuntu uwsgi[1717]: spawned uWSGI worker 5 (pid: 1726, cores: 1)
Feb 27 13:25:07 ubuntu uwsgi[1717]: Tue Feb 27 13:25:07 2018 - [emperor] vassal bl-status-api.ini has been spawned
Feb 27 13:25:07 ubuntu uwsgi[1717]: Tue Feb 27 13:25:07 2018 - [emperor] vassal bl-status-api.ini is ready to accept requests

```

