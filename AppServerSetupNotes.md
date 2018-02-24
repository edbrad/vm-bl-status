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
[uwsgi]
project = sitename
uid = netadmin
base = /home/%(uid)

chdir = %(base)/%(project)
home = %(base)/Env/%(project)
module = %(project).wsgi:application

master = true
processes = 5

socket = /run/uwsgi/%(project).sock
chown-socket = %(uid):www-data
chmod-socket = 660
vacuum = true
```
