# Online MTA-STS testing tool

This tool checks MTA-STS (RFC 8461), SMTP TLS reporting (TLSRPT), and experimental DANE configuration for a mail domain. MTA-STS is a standardized mechanism for protecting SMTP connections against downgrade attacks. This tool is not a comprehensive mail-security audit; verify important results independently.

## Prerequisites

- Ubuntu Server 24.04 or 26.04 LTS and an account with `sudo` privileges.
- Allow incoming TCP connections on port 80; allow port 443 as well if HTTPS is configured.
- Allow outgoing DNS queries over UDP and TCP port 53. DNS nameservers are configured in `settings.json`; the example configuration uses Quad9 (`9.9.9.9`, `149.112.112.112`, `2620:fe::fe`, and `2620:fe::9`).
- Allow outgoing HTTPS connections over TCP port 443 to retrieve MTA-STS policy files and install dependencies.
- Allow outgoing SMTP connections over TCP port 25 to the mail servers being checked. Some hosting providers block outbound SMTP; mail-server checks may fail or time out if this port is unavailable.
- Keep the uWSGI application socket (`127.0.0.1:17000`) and stats endpoint (`127.0.0.1:17005`) bound to localhost; do not expose them through the firewall.

## Installing application on Ubuntu 24.04 or 26.04 LTS

1. Install dependencies:
    ```bash
    sudo apt update
    sudo apt install -y uwsgi uwsgi-plugin-python3 python3-flask python3-dnspython python3-venv

    sudo python3 -m venv --system-site-packages /opt/mta-sts-venv
    sudo /opt/mta-sts-venv/bin/python -m pip install Flask-Limiter
    ```

2. Create a configuration file for uWSGI at `/etc/uwsgi/apps-available/emperor.ini`:

    ```bash
    sudo tee /etc/uwsgi/apps-available/emperor.ini > /dev/null <<'EOF'
    [uwsgi]
    emperor = /etc/uwsgi/vassals

    uid = www-data
    gid = www-data

    die-on-term = true
    logto = /var/log/uwsgi/emperor.log
    EOF
    ```

    ```bash
    sudo chmod 644 /etc/uwsgi/apps-available/emperor.ini
    sudo ln -s /etc/uwsgi/apps-available/emperor.ini /etc/uwsgi/apps-enabled/emperor.ini
    ```
	
3. Create `/etc/uwsgi/vassals/`:
 
    ```bash
    sudo install -d -m 0755 /etc/uwsgi/vassals
    ```

4. Create the log directory:
    ```bash
    sudo install -d -o www-data -g www-data -m 0755 /var/log/uwsgi
    ```

5. Create a configuration for this app at `/etc/uwsgi/vassals/mta-sts.ini`:

    ```bash
    sudo tee /etc/uwsgi/vassals/mta-sts.ini > /dev/null <<'EOF'
    [uwsgi]
    uid                = www-data
    gid                = www-data
    cap                = setgid,setuid

    socket             = 127.0.0.1:17000

    manage-script-name = true
    mount              = /=check:app

    plugins            = python3
    pythonpath         = /var/www/html/mta-sts
    virtualenv         = /opt/mta-sts-venv

    chmod-socket       = 660

    stats              = 127.0.0.1:17005

    processes          = 1

    pidfile            = /run/uwsgi/%n.pid
    logto              = /var/log/uwsgi/mta-sts.log

    vacuum             = true
    die-on-term        = true
    EOF
    ```

    ```bash
    sudo chmod 644 /etc/uwsgi/vassals/mta-sts.ini
    ```
6. Replace the package's SysV service with a native systemd unit:

   Ubuntu's uWSGI package provides `/etc/init.d/uwsgi`, which systemd exposes as a generated service. The unit below at `/etc/systemd/system/uwsgi.service` takes precedence over that generated service. On an existing installation, make sure no old Emperor or Vassal processes remain and ports `17000` and `17005` are free before starting this unit.

    ```bash
    sudo tee /etc/systemd/system/uwsgi.service > /dev/null <<'EOF'
    [Unit]
    Description=uWSGI Emperor
    After=network-online.target
    Wants=network-online.target

    [Service]
    Type=simple

    ExecStartPre=/usr/bin/install -d -o www-data -g www-data -m 0755 /run/uwsgi
    ExecStart=/usr/bin/uwsgi --ini /etc/uwsgi/apps-enabled/emperor.ini

    Restart=on-failure
    RestartSec=5s

    KillSignal=SIGTERM
    KillMode=mixed
    TimeoutStopSec=90s
    SendSIGKILL=yes

    StandardOutput=journal
    StandardError=journal

    [Install]
    WantedBy=multi-user.target
    EOF
    ```

7. Install the application:
	
    ```bash
    sudo git clone --branch customize https://github.com/wydler/mta-sts /var/www/html/mta-sts
    sudo cp /var/www/html/mta-sts/settings.example.json /var/www/html/mta-sts/settings.json
    sudo chmod 644 /var/www/html/mta-sts/settings.json
    ```

    Edit `/var/www/html/mta-sts/settings.json` to configure the DNS nameservers and, if needed, the SMTP local hostname. This local file is ignored by Git, so its settings are retained when updating the application.

8. Enable and start the service:

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable --now uwsgi.service
    sudo systemctl restart uwsgi.service
    sudo systemctl status uwsgi.service

    sudo ss -ltnp | grep -E ':17000|:17005'
    ```

9. Install `uwsgitop` system-wide:

    ```bash
    sudo python3 -m venv /opt/uwsgitop
    sudo /opt/uwsgitop/bin/python -m pip install uwsgitop
    sudo ln -s /opt/uwsgitop/bin/uwsgitop /usr/local/bin/uwsgitop
    ```

    To upgrade `uwsgitop` later:
    ```bash
    sudo /opt/uwsgitop/bin/pip install --upgrade uwsgitop
    ```

    Start `uwsgitop` to monitor the uWSGI stats endpoint:

    ```bash
    uwsgitop 127.0.0.1:17005
    ```

		
## Choose a web server

Choose either Apache or nginx. The following sections are alternatives; use only the section for the web server you want.

### Installing apache2 on Ubuntu 24.04 or 26.04 LTS

1. Install webserver and dependencies:

    ```bash
    sudo apt install -y apache2 apache2-dev
    wget https://raw.githubusercontent.com/unbit/uwsgi/2.0.31/apache2/mod_proxy_uwsgi.c
    sudo apxs2 -i -c mod_proxy_uwsgi.c
    sudo tee /etc/apache2/mods-available/proxy_uwsgi.load > /dev/null <<'EOF'
    LoadModule proxy_uwsgi_module /usr/lib/apache2/modules/mod_proxy_uwsgi.so
    EOF
    sudo a2enmod proxy proxy_uwsgi
    ```

2. Expand the global Apache configuration and the virtual host configuration. The `LoadModule` directive is enabled globally by `a2enmod` above; do not put it inside a `<VirtualHost>` block. Add the remaining lines to `/etc/apache2/sites-available/000-default.conf`, for example:

    ```apache2
    <VirtualHost *:80>
        ...
        DocumentRoot /var/www/html/mta-sts
        ...
        ProxyPass /api uwsgi://127.0.0.1:17000
        ProxyPassReverse /api uwsgi://127.0.0.1:17000
        ...
    </VirtualHost>
    ```

3. Restart service:
 
    ```bash
    sudo apache2ctl configtest && sudo systemctl restart apache2.service
    ```


### Installing nginx on Ubuntu 24.04 or 26.04 LTS

1. Install webserver:

    ```bash
    sudo apt install -y nginx
    ```

2. Expand existing webserver configuration with these lines. For example `/etc/nginx/sites-enabled/default`:

    ```nginx
    server {
        ...
        root /var/www/html/mta-sts;
        ...
        index index.html;
        ...
        location = /api {
                include uwsgi_params;
                uwsgi_pass 127.0.0.1:17000;
        }
        ...
    }
    ```
3. Restart service:
 
    ```bash
    sudo nginx -t && sudo systemctl restart nginx.service
    ```
		

## Testing

Open `http://<IP address or FQDN>/` (or use `https://` if configured) and submit a domain to check.

To test the API directly through nginx, run:

```bash
curl -N -v --max-time 360 'http://127.0.0.1/api?domain=example.com'
```

If the webserver is configured for a specific hostname or HTTPS, use that hostname and scheme instead of `127.0.0.1`. A successful request returns `HTTP 200`, a `text/event-stream` content type, and streamed lines beginning with `data:`.

## Troubleshooting

If the API returns an error or does not respond, check the service state and logs:

```bash
sudo systemctl status --no-pager -l uwsgi.service
sudo journalctl -u uwsgi.service -n 100 --no-pager
sudo tail -n 100 /var/log/uwsgi/emperor.log
sudo tail -n 100 /var/log/uwsgi/mta-sts.log
sudo tail -n 100 /var/log/nginx/error.log
sudo ss -ltnp | grep -E ':17000|:17005'
```
