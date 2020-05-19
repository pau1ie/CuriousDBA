---
title: "Django on RedHat 7"
date: 2020-03-06T0:03:50Z
tags: ['Apache','Linux','Django','Python','Postgresql','HTTP','REST']
---

# Django on RHEL7

Here is how I chose to set up Django on RHEL7. This is somewhat painful because the default
Python in RHEL7 is still version 2. Using this to create any new programs seems foolish as it
has already been de-supported by the Python project, so I really have to use Python 3.


## Set up Linux

We need Python 3, Django, Apache, and Postgresql. See the instructions linked in the next section
if the Postgresql packages aren't found. The version that comes with RHEL7 is too old for
the version of Django we want to install.

```bash
yum install httpd python3 python3-devel httpd-devel \
            postgresql12 postgresql12-server python3-psycopg2
pip3 install django djangorestframework Markdown
pip3 install mod_wsgi
```

# Create the database.

Postgresql has instructions for 
[installing on RedHat](https://www.postgresql.org/download/linux/redhat/).

```bash
/usr/pgsql-12/bin/postgresql-12-setup initdb
systemctl enable postgresql-12.service
systemctl start postgresql-12.service
```

## Set up the database

```console
# su - postgres
$ createuser --pwprompt myuser
Enter password for new role: 
Enter it again: 

$ createdb -O myuser mydb "Environment list database"
```

Make sure the database accepts passwords:

Edit `/var/lib/pgsql/data/pg_hba.conf` and change the following lines:

```
host    all             all             127.0.0.1/32            ident
host    all             all             ::1/128                 ident
```
To:
```
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

Reload the service:

```bash
systemctl restart postgresql-12.service
```

# Set up Django

The [Django website](https://docs.djangoproject.com/en/3.0/howto/deployment/wsgi/uwsgi/)
has instructions on how to set up Django to run under wsgi.

## Create the Project Directory

```bash
cd /var/local
mkdir -p www/django
cd www/django
django-admin startproject myapp
```


## Set up urls.py

Add the URL to the `urls.py`, and add the required imports as follows:

```python {hl_lines=[2,3,6,7]}
from django.contrib import admin
from django.urls import path, include
from envs import views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('envs/', include('envs.urls')),
]
```


## Edit settings.py

Add the project and restframework to the `INSTALLED_APPS` variable.

```python {hl_lines=["8-10"]}
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'rest_framework.authtoken',
    'envs',
]
```

Also add TokenAuthentication to the `REST_FRAMEWORK` setting:

```python
    REST_FRAMEWORK = {
        'DEFAULT_AUTHENTICATION_CLASSES': [
            'rest_framework.authentication.TokenAuthentication',
        ],
    }
```

Change the database to use postgresql by replacing the `DATABASES` definition
with the following:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'password',
        'HOST': 'localhost',
    }
}
```

The `HOST: Localhost` makes the connection  use the host, because local connections are set
(in `pg_hba.conf`) to identify as peer, which means they have the same username in the database as in the OS.

Add the location for the static files. These *just work* in debug mode with `runserver`, but
need to have a place to be served from when running from Apache:

```python {hl_lines=[5]}
# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/3.0/howto/static-files/

STATIC_URL = '/static/'
STATIC_ROOT="/var/local/www/django/static"
```

Another change required is the `ALLOWED_HOSTS` line:

```python
ALLOWED_HOSTS=['hostname','fully.qualified.hostname']
```

Where *hostname* the hostname of the VM that is running Django. 

Lastly, take the Django out of debug mode.

```python
DEBUG = False
```

## Run required utilities

Run the migrations:

```bash
python3 manage.py migrate
```

Build  the static files directory:
```console
python3 manage.py collectstatic

163 static files copied to '/var/local/www/django/static'.
```

I check here if the environment works.

```
python3 manage.py runserver 0.0.0.0:8000
```

This isn't suitable for production use though, I need to set up Apache.

## Set up the Django Application Users

```console
# python3 manage.py createsuperuser
Username (leave blank to use 'root'): me
Email address: me@cam.ac.uk
Password: 
Password (again): 
Superuser created successfully.
```

Now I can log in to my Django website, and with this username I can create any other users I require.

I am using token authentication for the API, so I need to create a token for the user.

```bash
python3 manage.py drf_create_token username
```

The token is echoed out to the terminal. This can be copied to the configuration of the API consumer.

# Set up Apache

The [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-serve-django-applications-with-apache-and-mod_wsgi-on-centos-7) 
documentation is really useful here as is the 
[official](https://docs.djangoproject.com/en/3.0/howto/deployment/wsgi/) 
[Django](https://docs.djangoproject.com/en/3.0/howto/deployment/wsgi/modwsgi/)
documentation.


## WSGI

Apache uses [WSGI](https://wsgi.readthedocs.io/en/latest/what.html) to run 
Python processes. It seems that there is a Python 2 version 
already installed in the operating system. I couldn't
work out where it was from, but in any case the new one needed to be configured as follows:

```bash
mod_wsgi-express install-module> /etc/httpd/conf.modules.d/02-wsgi.conf
```

The configuration for the old version was removed as follows:

```bash
rm /etc/httpd/conf.modules.d/10-wsgi.conf
```

## Apache Configuration

An Apache configuration for Django was created by creating `/etc/httpd/conf.d/django.conf` with
the following contents:

```ApacheConf
Alias /static /var/local/www/django/static
<Directory  /var/local/www/django/static>
  Require all granted
</Directory>

<Directory /var/local/www/django/myapp/>
  Require all granted
</Directory>

<Directory /var/local/www/django/myapp/myapp>
  <Files wsgi.py>
    Require all granted
  </Files>
</Directory>

WSGIDaemonProcess myapp python-path=/var/local/www/django/myapp:/usr/local/lib64/python3.6/site-packages
WSGIProcessGroup myapp
WSGIScriptAlias / /var/local/www/django/myapp/myapp/wsgi.py
WSGIPassAuthorization On
```

The static directory is the one that was built earlier using `collectstatic`.

Note that I haven't used a virtual environment. Doing so seems quite straightforward, so this might
have been a better thing to do. As things stand I had to add the location where Django and the
rest framework were installed to the python-path. Should Python  be upgraded this will need to change.

A quick test shows this runs on Apache.


## Using a self signed certificate.

This VM is inside a NATted nework. Nothing on the internet can see us, so we can't get a
[Let's Encrypt](https://letsencrypt.org/) certificate. To make sure passwords to login to the
admin site aren't sent in the clear over the network, I will use a self-signed certificate.

There was already a self signed certificate, so just pointing the browser at the website using
https worked. I had to accept the warnings that the certificate authority was untrusted.
Also the certificate had expired. My understanding is that this means the traffic is encrypted
over the network, but we can't use the certificate to protect from man in the middle attacks, 
i.e. it won't verify the server the request comes from. Since this is an internal network I
think I am happy with this.

## Prevent unencrypted traffic

This requires mod_rewrite which was already installed. I just had to add the following
file: `/etc/httpd/conf.d/http.conf`

```ApacheConf
<VirtualHost *:80>
ServerName fully.qualified.domain.name
Redirect permanent / https://fully.qualified.domain.name/
</VirtualHost>
```

# Conclusion

This turned out to be a lot of effort, but now I have an application hosted that uses JSON REST APIs.
This can be used by various scripts to keep information about the environments I maintain up to date.