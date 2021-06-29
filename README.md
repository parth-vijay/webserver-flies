# Set up Webinar project on Ubuntu.

## SSH to instance where you want to deploy your project:
	ssh -i <key.pem> <username>@<ipaddress>

## Create new user in instance:
	sudo usermod -m webinar

## Create Python directory inside user home directory and give 777 permission to it:
	cd /home/webinar
	sudo mkdir python
	sudo chmod -R 777 python

## Change to python directory and crete python environment:
	cd /home/webinar/python
	python -m venv env

## Activate the environment:
	source env/bin/activate

## Clone you repository inside python directory:
	cd /home/webinar/python
	git clone <repo-url>
	git clone https://github.com/vsjakhar/webinar.git

### Note (Only for webinar): In webinar project, we create the copy of project for each domain. Also, we have separate branch for each domain we deploy. For instance, for domain https://ibs21.pueretevisual.com we have branch named 'ibs21'.

## Create the copy of project for required domain:
	cp -r /home/webinar/python/webinar  /home/webinar/python/ibs21
	cd /home/webinar/python/ibs21
	git checkout ibs21

## Install required python modules:
	cd /home/webinar/python/ibs21
	pip3 install -r requirements.txt
	pip install gunicorn gevent  # Will be used in Gunicorn service configuration 

# Configure Database for project.

## Login to the database console(MySQL):
	mysql -h <host-name or ip-address> -u <username> -p
	create database <db_name>;
	create user '<user>'' identified by '<passowrd>';
	grant all privileges on <db_name>.* to '<user>'@'%';

## Update your django setting file in Django for configuring database:
	DATABASES = {
	    'default': {
	         'ENGINE': 'django.db.backends.mysql',
	         'NAME': '<db_name>',
	         'USER': '<username>',
	         'PASSWORD': '<password>',
	         'HOST': '<host-name or ip-address>',
	         'PORT': '3306',
	    }
	}

## Run the migrations commands:
	./manage.py makemigrations
	./mange.py migrate

## Configuring Static files on S3 or Azure Blob Storage:

**For AWS**
	Install following modules:-

	pip install boto3 django-storages==1.11.1

Update your django setting file:-

	AWS_ACCESS_KEY_ID = <AWS-ACCESS-KEY-ID>
	AWS_SECRET_ACCESS_KEY = <AWS-SECRET-ACCESS-KEY>
	AWS_STORAGE_BUCKET_NAME = <S3-BUCKET-NAME>
	AWS_S3_CUSTOM_DOMAIN = '%s.s3.amazonaws.com' % AWS_STORAGE_BUCKET_NAME
	AWS_S3_OBJECT_PARAMETERS = {
	    'CacheControl': 'max-age=86400',
	    'ACL':'public-read'
	}
	AWS_LOCATION = 'static'
	AWS_DEFAULT_ACL = None
	STATIC_URL = 'https://%s/%s/' % (AWS_S3_CUSTOM_DOMAIN, AWS_LOCATION)
	STATICFILES_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
	DEFAULT_FILE_STORAGE = 'webinarwall.storage_backends.MediaStorage'

Create & configure *storage_backends.py* file in your project directory:-

	from storages.backends.s3boto3 import S3Boto3Storage
	from django.conf import settings
	class MediaStorage(S3Boto3Storage):
	    location = 'media'
	    file_overwrite = False


**For Azure**
	Install following modules:-

	pip install boto3 django-storages==1.11.1 django-storages-azure==1.6.8 azure-storage==0.36.0

Update your django setting file:-

	AZURE_ACCOUNT_KEY = <AZURE-ACCOUNT-KEY>
	AZURE_ACCOUNT_NAME = <AZURE-ACCOUNT-NAME>
	AZURE_CONTAINER = <AZURE-CONTAINER-NAME>
	AZURE_CUSTOM_DOMAIN = f'{AZURE_ACCOUNT_NAME}.blob.core.windows.net'
	STATIC_URL = f'https://{AZURE_CUSTOM_DOMAIN}/static/'
	MEDIA_URL = f'https://{AZURE_CUSTOM_DOMAIN}/media/'

Create & configure *storage_backends.py* file in your project directory:-

	from storages.backends.s3boto3 import S3Boto3Storage
	from storages.backends.azure_storage import AzureStorage
	from django.conf import settings
	class AzureMediaStorage(AzureStorage):
	    account_name = settings.AZURE_ACCOUNT_NAME
	    account_key = settings.AZURE_ACCOUNT_KEY
	    azure_container = 'media'
	    expiration_secs = None
	class AzureStaticStorage(AzureStorage):
	    account_name = settings.AZURE_ACCOUNT_NAME
	    account_key = settings.AZURE_ACCOUNT_KEY
	    azure_container = 'static'
	    expiration_secs = None
	    file_overwrite = True

## Run collectstatic command:
	./manage.py collectstatic
	./manage.py collectstatic --no-input

## Configure Webserver to host your app:
**Apche2**
Installing on Fedora/CentOS/Red Hat Enterprise Linux:-

	sudo yum install httpd
	sudo systemctl enable httpd
	sudo systemctl start httpd
	sudo yum install mod_wsgi

Installing on Ubuntu/Debian:-

	sudo apt install apache2
	sudo service apache2 start
	sudp apt install libapache2-mod-wsgi libapache2-mod-wsgi-py3 

Installing on Mac OS:-

	xcode-select --install
	brew install httpd
	brew services start httpd
	brew install mod_wsgi

Configure Apache2 Virtualhost file (/etc/apache2/sites-avavilable/test.conf):-

	<VirtualHost 0.0.0.0:80>
		######   Redirect HTTP to HTTPS  #######
        ServerName testdomain.com
        ServerAlias www.testdomain.com

        Redirect permanent / https://testdomain.com/
	</VirtualHost>

	<VirtualHost 0.0.0.0:443>
        ServerAdmin support@gmail.com
        ServerName testdomain.com
        ServerAlias www.testdomain.com
        DocumentRoot /home/webinar/python/<project-directory-name>

        ######   Configure Chat server    ######## 
        RewriteEngine   On
        RewriteCond %{HTTP:Upgrade}             =websocket                                      [NC]
        RewriteRule ^/ws/(.*)                   ws://0.0.0.0:9999/ws/$1         [P,L]

        ######   SSL Certificate   ######
        SSLEngine      on
        SSLCertificateFile       <path-to-certificate-pem-file>
        SSLCertificateKeyFile  	 <path=to=certificate-private-key-file>

        ######   Configuration for Static Files   ######
        Alias /static /home/webinar/python/<project-directory-name>/static
        <Directory /home/webinar/python/<project-directory-name>/static>
                Require all granted
        </Directory>

        ######  Configuration for Media files   #######
        Alias /media /home/webinar/python/<project-directory-name>/media
        <Directory /home/webinar/python/<project-directory-name>/media>
                Require all granted
        </Directory>

        <Directory /home/webinar/python/<project-directory-name>/<django-project-name>>
                <Files wsgi.py>
                        Require all granted
                </Files>
        </Directory>

        WSGIApplicationGroup %{GLOBAL}
        WSGIDaemonProcess test python-home=<path-to-python-env> python-path=<path-to-your-project-directory>
        WSGIProcessGroup test
        WSGIPassAuthorization On
        WSGIScriptAlias / <path-to-django-project-wsgi-file>
	</VirtualHost>

**Nginx**
Installing on Fedora/CentOS/Red Hat Enterprise Linux:-

	sudo yum install epel-release
	sudo yum update
	sudo yum install nginx
	sudo nginx -v
	sudo service start nginx

Installing on Ubuntu/Debian:-

	sudo apt install nginx
	sudo service start nginx

Installing on Mac OS:-

	brew install nginx
	sudo nginx

Configure Nginx Virtualhost file (/etc/nginx/sites-avavilable/test.conf):-

	proxy_cache_path /tmp/cache1 keys_zone=cache1:10m levels=1:2 inactive=600s max_size=1g use_temp_path=off;

	upstream websocket {
	    server 127.0.0.1:8111;
	}

	######   Redirect HTTP to HTTPS  #######
	server {
	    listen 80;
	    server_name testdomain.com  www.testdomain.com;
	    return 301 https://testdomain.com;
	}

	server {
		######   SSL Certificate   ######
	    listen 443;
	    ssl on;
	    ssl_certificate   <path-to-certificate-pem-file>
	    ssl_certificate_key <path=to=certificate-private-key-file>
	    server_name testdomain.com  www.testdomain.com;

	    root /home/webinar/python/<project-directory-name>;

	    #access_log /var/www/logs/access.log;
	    error_log  /var/www/logs/error.log;

	    location / {
	        include proxy_params;
	        proxy_read_timeout 300s;
	        proxy_connect_timeout 300s;
	        proxy_pass http://unix:/opt/run/test.sock;
	        proxy_cache cache1;
	        proxy_cache_revalidate on;
	        proxy_cache_min_uses 3;
	        proxy_cache_valid any 1m;
	        proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504 updating;
	    }

	    location /ws/ {
	        proxy_pass http://websocket;
	        proxy_http_version 1.1;
	        proxy_set_header Upgrade $http_upgrade;
	        proxy_set_header Connection "upgrade";
	        proxy_read_timeout 86400;
	    }

	    ######   Configuration for Static Files   ######
	    location /static/ {
	        alias /home/webinar/python/<project-directory-name>/static/; # ending slash is required
	    }

	    ######   Configuration for Media Files   ######
	    location /media/ {
	        alias /home/webinar/python/<project-directory-name>/media/; # ending slash is required
	    }
	}

Set up Gunicorn service file (/etc/systemd/system/gunicorn_test.service):-
	
**You can give any name to your service file but the extension(.service) is must.
Also, Gunicorn service file will be configured separately for each Nginx virtualhost file.**

	[Unit]
	Description=gunicorn daemon
	After=network.target

	[Service]
	User=ubuntu
	Group=www-data
	WorkingDirectory=/home/webinar/python/<project-directory-name>
	ExecStart=/home/webinar/python/env/bin/gunicorn --access-logfile - --timeout 300 --worker-class gevent --workers 1   --backlog 50 --worker-connections=100 --bind unix:/opt/run/test.sock <django-project-name>.wsgi:application

	[Install]
	WantedBy=multi-user.target

Activate the gunicorn service file:
	
	sudo systemctl daemon-reload
	sudo systemctl start gunicorn_test
	sudo systemctl enable gunicorn_test
	sudo systemctl status gunicorn_test
	sudo systemctl restart gunicorn_test


## Install Letsencrypt for free SSL certificate:

Installing on Fedora/CentOS/Red Hat Enterprise Linux:-

	sudo yum -y install epel-release
	sudo yum -y install yum-utils
	sudo yum -y install certbot-apache  (For Apache2)
	sudo dnf install certbot python3-certbot-nginx  (For Nginx)

Installing on Ubuntu/Debian:-

	sudo apt install software-properties-common
	sudo add-apt-repository universe
	sudo apt update
	sudo apt install certbot python3-certbot-apache  (For Apache2)
	sudo apt install certbot python3-certbot-nginx  (For Nginx)

**Note:- Make sure you have created your DNS record for your domain.**
To check whether the IP has been pointed to your domain, run the following command:-

	nslookup <your-domain-name>

Issue the SSL certifictae:-

	sudo certbot certonly -d <your-domain-name>

## Enable the Virtualhost file and restart the web server:
	
**Apache2**

	sudo a2ensite test.conf
	sudo systemctl reload apache2
	sudo systemctl enable apache2
	sudo service apache2 restart

**Nginx**

	sudo ln -s /etc/nginx/sites-available/test.conf  /etc/nginx/sites-enabled/
	sudo systemctl restart nginx
	sudo service nginx restart

## Configure Chat server:

Installing Redis server on Fedora/CentOS/Red Hat Enterprise Linux:-

	sudo yum install epel-release
	sudo yum install redis -y
	sudo systemctl start redis.service
	sudo systemctl enable redis
	sudo systemctl status redis.service

Installing Redis server on Ubuntu/Debian:-
	
	sudo apt update
	sudo apt install redis-server
	sudo systemctl start redis
	sudo systemctl enable redis
	sudo systemctl status redis

Installing Redis server on Mac OS:-

	brew install redis
	brew services start redis
	brew services status redis

Install the python module required for chat server:-

	pip3 install daphne

Configure service file for web socket or chat server (/etc/systemd/system/daphne_test.service):-

**You can give any name to your service file but the extension(.service) is must.
Also, Daphne service file will be configured separately for each project or for each nginx virtualhost file.**

	[Unit]
	Description=daphne daemon for web socket
	After=network.target

	[Service]
	User=ubuntu
	Group=www-data
	WorkingDirectory=/home/webinar/python/<project-directory-name>
	ExecStart=/home/webinar/python/env/bin/daphne -b 0.0.0.0 -p 8111 <django-project-name>.asgi:application

	[Install]
	WantedBy=multi-user.target

**Note:-** Port number should be different for each number and should be same as given in Nginx or Apache2 virtualhost file.

Activate the daphne service file:

	sudo systemctl daemon-reload
	sudo systemctl start daphne_test
	sudo systemctl enable daphne_test
	sudo systemctl status daphne_test



# Set up Angular project on server.

## Create Apache configuration file for angular app (/etc/apache/sites-available/angular.conf):

	<VirtualHost *:80>
		ServerAdmin support@example.com
		ServerName myangularapp.com
		ServerAlias www.myangularapp.com
		DocumentRoot /var/www/dist
		<Directory “/var/www/dist”>
			AllowOverride All
			Require all granted
		</Directory>
	</VirtualHost>

## Enable angular.conf file using the following command:

	a2ensite angular.conf

## Restart the apache service:

	sudo systemctl restart apache2