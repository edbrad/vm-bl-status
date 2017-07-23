# vm-bl-status
The purpose of this repository is to document the set-up and configuration of the application server for the bl-status (Box Loading Status) system.

## Overview
The Box Loading [bl-status] system consists of a database of status information, a REST API that provides external access to the database, and a front-end GUI web application that allows users to manage the status. These three major components are hosted on a single virtual machine server. The following *Open Source* products and technologies are being used:

* **Ubuntu Linux [16.04 LTS]:** Host Operating System (http://www.ubuntu.com)
* **MongoDB [version 3.4]:** NoSQL Database server (http://mongodb.org)
* **Django w/ REST Framework:** REST API Server & Administrative Web Application (http://djangoproject.com)
* **Nginx:** Reverse Proxy / Web Server (http://nginx.org)
* **Python [version 3]:** Popular and powerful Programming/Scripting Language. Underlying language for Django (http://python.org)
* **uWSGI:** Web Site Gateway Interface for Python-based applications (Django) - (http://uwsgi-docs.readthedocs.io/en/latest/)
* **Angular [version 4]:** Front-end Javascript Framework. Used to develop the GUI Web application (http://angular.io)


![vm-bl-status diagram](./images/vm-bl-status_server.png)

