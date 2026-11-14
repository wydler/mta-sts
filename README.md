# Online MTA-STS testing tool

This tool verifies whether a give host correctly implements the new in-development <a href="https://github.com/mrisher/smtp-sts">MTA-STS standard</a> for downgrade-resistant secure email. It is very new and not very well tested so don't rely on it's result too much.

Online version: https://mta-sts-tester.cl01.daniel.wydler.eu/

License: BSD 2-clause license (see LICENSE.txt).

## Installing application on Ubuntu 24.04 LTS

 1. Install dependencies:
    ```bash
    apt update
    apt install -y uwsgi uwsgi-plugin-python3 python3-flask python3-dnspython python3-venv pipx

    python3 -m venv --system-site-packages /opt/mta-sts-venv
    /opt/mta-sts-venv/bin/python -m pip install Flask-Limiter
    ```

 2. Create a configuration file for uWSGI at `/etc/uwsgi/apps-available/emperor.ini`:

    ```ini
    [uwsgi]
	emperor = /etc/uwsgi/vassals
	uid = www-data
	gid = www-data
	limit-as = 512
	reload-on-as = 128
	reload-on-rss = 192
	logto = /tmp/uwsgi.log
    ```
    ```bash
    chmod 644 /etc/uwsgi/apps-available/emperor.ini
    cd /etc/uwsgi/apps-enabled
    ln -s ../apps-available/emperor.ini .
    cd ~
    ```
	
 3. Create `/etc/uwsgi/vassals/`:
 
    ```bash
    mkdir /etc/uwsgi/vassals
    chmod 755 /etc/uwsgi/vassals
    ```
	
 4. Create a configuration for this app at `/etc/uwsgi/vassals/mta-sts.ini`:

    ```ini
	[uwsgi]
	uid                = www-data
	gid                = www-data
	cap                = setgid,setuid
	socket             = 127.0.0.1:17000
	manage-script-name = true
	mount              = /=check:app
	plugins            = python3
	chmod-socket       = 660
	pythonpath         = /var/www/html/mta-sts
	virtualenv         = /opt/mta-sts-venv
	stats              = 127.0.0.1:17005
	vacuum             = True
	processes          = 1
	pidfile            = /tmp/%n.pid
	logto              = /tmp/mta-sts-uwsgi.log
	die-on-term        = true
    ```
    ```bash
    chmod 644 /etc/uwsgi/vassals/mta-sts.ini
    ```

 5. Restart service:
 
    ```bash
    systemctl restart uwsgi.service
    ```

 6. Install the application:
	
    ```bash
    git clone --branch customize https://github.com/wydler/mta-sts /var/www/html/mta-sts
    ```
		

 7. Install a monitoring tool for it as the user who will run it:

    ```bash
    pipx ensurepath
    pipx install uwsgitop
    ```

    Open a new shell if `pipx ensurepath` changed your `PATH`, then run:

    ```bash
    uwsgitop 127.0.0.1:17005
    ```

		
## Installing apache2 on Ubuntu 24.04 LTS

 1. Install webserver and dependencies:

    ```bash
    apt install -y apache2 apache2-dev
    wget https://github.com/unbit/uwsgi/raw/master/apache2/mod_proxy_uwsgi.c
    apxs2 -i -c mod_proxy_uwsgi.c
    a2enmod proxy_http
    ```

 2. Expand existing webserver configuration with these lines. For example `/etc/apache2/sites-available/000-default.conf`:

    ```apache2
	<VirtualHost *:80>
	...
	DocumentRoot /var/www/html/mta-sts
	...
	LoadModule proxy_uwsgi_module /usr/lib/apache2/modules/mod_proxy_uwsgi.so
	ProxyPass /api uwsgi://127.0.0.1:17000
	ProxyPassReverse /api uwsgi://127.0.0.1:17000
	...
	</VirtualHost>
    ```

 3. Restart service:
 
    ```bash
    systemctl restart apache2.service
    ```


## Installing nginx on Ubuntu 24.04 LTS

 1. Install webserver:

    ```bash
    apt install -y nginx
    ```

 2. Expand existing webserver configuration with these lines. For example `/etc/nginx/sites-enabled/default`:

    ```nginx
    server {
        ...
        root /var/www/html/mta-sts/;
        ....
        location = /api {
                include uwsgi_params;
                uwsgi_pass 127.0.0.1:17000;
        }
        ...
    }
    ```
 3. Restart service:
 
    ```bash
    systemctl restart nginx.service
    ```
		

## Remark
  Don't forget to test, that the applcation works sucessully.
  Open `http://<IP address or FQDN>/mta-sts/` (or use `https://` if configured).
