---
title: "Django on RedHat 9"
date: 2024-03-12T17:03:50+00:00
tags: ['Apache','Linux','Django','Python','Postgresql','HTTP','REST']
---

# Django on RHEL9

The reason I am revisiting my Django website is because I am updating
the operating system to RHEL9. At the same time I decided to take the 
opportunity to correct a couple of things I was unhappy with in the 
previous setup.

I prefer to take a clean VM and build from that. That way we know
the exact configuration and can document it.


## Set up Linux

We need to install the following (Using YUM)

 * python3
 * python3-devel
 * httpd (i.e. Apache)
 * postgresql
 
 And the following are installed with pip
 
  * Django
  * social-auth-app-django

# Create the database.

Postgresql has instructions for 
[installing on RedHat](https://www.postgresql.org/download/linux/redhat/).
However my administrator has already installed postgres for me, so I just used 
that installation. The difference is that the service is called postgresql 
without a version number.


## Set up the database

Make sure the database accepts passwords. The documentation suggests we 
should be using sha-256 rather than md5 for passwords, so we can configure 
them as follows:

Edit `/var/lib/pgsql/data/postgresql.conf` and ensure the password encryption 
is `scram-sha-256`:

```
password_encryption = scram-sha-256              # md5 or scram-sha-256
```

Edit `/var/lib/pgsql/data/pg_hba.conf` and change the following lines:

```
host    all             all             127.0.0.1/32            ident
host    all             all             ::1/128                 ident
```
To:
```
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ::1/128                 scram-sha-256
```

Reload the service:

```bash
systemctl restart postgresql
```

## Export and Import the Data

Since this is a pre-existing website, I chose to export and import the data. 
This is a tiny database, so exporting in SQL format is fine.

On the old server I ran:

```bash
su - postgres
pg_dump mydb > /tmp/my_db.sql
```

And on the new server I copied the file across, then ran the SQL into the 
database:

```bash
# Switch to postgres user
su - postgres
# Run postgres sql interface
psql
-- Create the mydb database and user
create database mydb;
CREATE USER myuser WITH PASSWORD 'password';
-- exit and switch to the new user
\q
psql -U myuser -d mydb -W -h localhost
-- Run the backup file
\i my_db.sql
exit
```


# Set up Django

## Create the Project User and Set Up Python

We need to create a new user. It's home directory will contain the django files, so
it will be under `/srv`. As root:

```bash
useradd appuser -G apache -d /srv/my_app --shell /bin/false
su - appuser --shell=/bin/bash
cd /srv/my_app
python3 -m venv myapp_env
. myapp_env/bin/activate
pip install --upgrade pip psycopg2
````

Having created the user we switch to it. We have to specify the shell as it's default shell
is `/bin/false`. Then we create the python environment, activate it and start to install some
dependencies.

I packaged up my app, and added requirements as follows in setup.cfg.

```
install_requires =
    Django >= 2.2.3
    djangorestframework >= 3.10.2
    Markdown >= 3.1.1
    social-auth-app-django
```

So I could install it as follows:

```
pip install my_app.tar.gz
```

I then created the Django environment as before:

```
mkdir my_app
cd my_app
django-admin startproject my_app
```

After this it is pretty much the same configuration as 
[before](../djangoonrhel7/), editing the 
[urls.py](../djangoonrhel7/#set-up-urlspy),
installed apps,
DEFAULT_AUTHENTICATION_CLASSES, the database and others in 
[settings.py](../djangoonrhel7/#edit-settingspy)


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

This VM is inside a NATted network. Nothing on the internet can see us, so we can't get a
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