# vm-bl-status
The purpose of this repository is to document the set-up and configuration of the application server for the bl-status (*Box Loading Status*) system.

## Overview
The Box Loading **[bl-status]** system consists of a database of Box Loading status information, a REST WebAPI that provides external access to the database (and any intermediate data processing), and a front-end GUI web application that allows users to manage the Box Loading status information. These primary components are hosted on a single server (virtual machine). The following *Open Source* products and technologies are implemented:

* **Ubuntu Linux [16.04 LTS]:** Host Operating System (http://www.ubuntu.com)
* **MongoDB [version 3.4]:** NoSQL Database server (http://mongodb.org)
* **Django w/ REST Framework:** REST API Server & Administrative Web Application (http://djangoproject.com)
* **Nginx:** Reverse Proxy / Web Server (http://nginx.org)
* **Python [version 3]:** Popular and powerful Programming/Scripting Language. Underlying language for Django (http://python.org)
* **uWSGI:** Web Site Gateway Interface for Python-based applications (Django) - (http://uwsgi-docs.readthedocs.io/en/latest/)
* **Angular [version 4]:** Front-end Javascript Framework. Used to develop the GUI Web application (http://angular.io)


![vm-bl-status diagram](./images/vm-bl-status_server.PNG)

## Server OS (Ubuntu Linux) Installation & Configuration
The system is hosted on a Virtual Machine *[VMWare]*.  The Server OS is **Ubuntu 16.04 LTS**, a widely-used and freely available Linux Operating system distribution.  A link to the installer image (ISO) is available here: https://www.ubuntu.com/download/server 

### OS Installation
The initial virtual hardware configuration is as shown bellow:

![vm config diagram](./images/vmconfig_page.PNG)

The Operating System is installed from the mounted ISO:

![ubuntu welcome](./images/ubuntu_welcome.PNG)

The server needs to connect to the company's **Proxy Server** [ISA-01] in order to download system updates through the Internet.  The installer provides a prompt for the address of the Proxy Server.  The current address is **[172.16.2.174:8080]**.  It is entered as shown below:

![ubuntu proxy server](./images/ubuntu_proxy_server.PNG)

Accessing and managing the server is much more convenint from an external terminal session.  This done over a secured/encrypted SSH connection.  During the installation the **OpenSSH server** option must be selected, as shown below:

![ubuntu openssh server](./images/ubuntu_OpenSSH_server.PNG)

### Install OS Updates
After the initial install, there will be system updates and security patches available to be installed. The Ubuntu Package Manager **(apt-get)** is used to retreive and install any available OS updates from the public online repsitories.  Enter the commands below to perform the update:
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
```
### Configure Networking
The TCP/IP Networking configuration must be adjusted to be beter suited for a Server role.  This includes: Assigning a Static IP address, specifing DNS Servers to be used for name resolution (Internet & EMSMAIL.COM), and enabling the Linux Firewall (w/ Ubuntu ufw) to protect the server from Network attacks.

### Install Remote Management GUI (Webmin)
**Webmin** is an popular Open Source Web-based Unix/Linux system management tool (http://webmin.com). It provides a comprehensive set of GUI tools to help with monitoring and performing common system management tasks without having to remember and type long commands into the console.  Installation requires adding the software's repository key, location and signature to the Ubuntu Package Manger. 

### Install MongoDB Database Server
