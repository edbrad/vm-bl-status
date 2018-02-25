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
```
[Unit]
Description=uWSGI Emperor service

[Service]
ExecStartPre=/bin/bash -c 'mkdir -p /run/uwsgi; chown netadmin:www-data /run/uwsgi'
ExecStart=/usr/local/bin/uwsgi --emperor /etc/uwsgi/sites
Restart=always
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all

[Install]
WantedBy=multi-user.target
```

## Install and Configure Nginx as a Reverse Proxy
The **Nginx** Web Server and Reverse Proxy sits in front the the uWSGI Application Server *(exposed to the public interface)*. Incoming requests and outgoing responses *(over http Port 80 in this case)* are routed by Nginx to and from the uWSGI Application Server via a **Unix socket** connection.  This socket-based internal communication has certain performance and security advantages over using http *(a http option is avaible, if needed)*.  As a *Reverse Proxy*, Nginx allows mulitple sites to be hosted on a single TCP Port *(80 in this case)*. It does this by examining the destination url in the incoming http request *(e.g. "bl-status-api.emsmail.com")* and routing the request to site that is configured to respond to that URL. 

**1. Install Nginx:**
```
$ sudo apt-get install nginx
```

**2. Create Nginx site configuration file:**
```
$ sudo nano /etc/nginx/sites-available/bl-status-api
```

**Nginx Site Configuration File Example:**
The site listens on port 80 (http) for requests directed at the URL for the API. Logging data is saved in the Linux Home folder for the autoritative User account.  An error will not be logged for Sites without an icon image file (favicon).  The Django folder containing the application's static assets is defined, along with the root entry folder for the application. The uWSGI Unix socket communication parameters are also specified:
```
server {
    listen 80;
    server_name bl-status-api.emsmail.com;
    
    access_log /home/netadmin/bl-status-logs/api/nginx_access.log;
    error_log /home/netadmin/bl-status-logs/api/nginx_error.log;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/netadmin/bl-status-api;
    }

    location / {
        include         uwsgi_params;
        uwsgi_pass      unix:/run/uwsgi/bl-status-api.sock;
    }
}
```

