# Installation

[fnndsc/pl-dircopy](https://github.com/FNNDSC/pl-dircopy) is required for the UI's _file upload_ feature.

"host" remote (not really) compute environment is hard-coded into _pfcon_.

# Required Plugins

- [fnndsc/pl-dircopy](https://github.com/FNNDSC/pl-dircopy) (is [hard-coded](https://github.com/FNNDSC/ChRIS_ui/blob/1efb856c2764d1f87bf077a44ee878f03e6bff1a/src/components/feed/CreateFeed/utils/createFeed.ts#L92) in [ChRIS_ui](https://github.com/FNNDSC/ChRIS_ui))

# Token Problems

Serving CUBE and ChRIS_ui on the same domain can cause session cookie collision, e.g. while logged into `/chris-admin/` you are barred from logging in to `/` (ChRIS_ui) unless you log out of `/chris-admin/` or delete the `sessionid` cookie.

Try serving CUBE and ChRIS_ui on different domains.

# Reverse Proxy

## Introduction

A _reverse proxy_ (e.g. nginx, Apache, [caddy](https://caddyserver.com/)) is usually used to enable HTTPS (and they also do load-balancing).

```
+----------------------+
| web browser          |
| e.g. firefox, chrome |
+----------------------+
      |
      | https://cube.example.com/api/vi/
      |
      V
+----------------------+                                 +----------------------+
| reverse proxy        |  http://localhost:8000/api/v1/  | backend              |
| e.g. nginx, apache   | ------------------------------> | ChRIS_ultron_backend |
+----------------------+                                 +----------------------+
```

## Internal Collection+JSON hrefs

`collection.links.href` is wrong.
Correct would be <code><b>https://</b>cube.example.com/api/v1/...</code> but you get something different:

- <code><b>http://</b>cube.example.com/api/v1/...</code>
- <code><b><span>http://</span>localhost:8000</b>/api/v1/...</code>

```json
{
  "collection": {
  "href": "http://cube.example.com/api/v1/",
  "items": [],
  "links": [
    {
      "href": "http://cube.example.com/api/v1/files/",
      "rel": "files"
    },
    {
      "href": "http://cube.example.com/api/v1/computeresources/",
      "rel": "compute_resources"
    }
  ],
  "total": 0,
  "version": "1.0"
}
```

Web browsers might block "mixed" requests, i.e. HTTP AJAX from an HTTPS origin.

### Fix

1. Configure your _reverse proxy_ to inject the [`X-Forwarded-Host` and `X-Forwarded-Proto` request headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Forwarded)
2. Add _django_ config options [`SECURE_PROXY_SSL_HEADER`](https://docs.djangoproject.com/en/dev/ref/settings/#secure-proxy-ssl-header) and [`USE_X_FORWARDED_HOST`](https://docs.djangoproject.com/en/dev/ref/settings/#use-x-forwarded-host) to `{chris,store}_backend/config/settings/production.py` in [fnndsc/ChRIS_ultron_backEnd](https://github.com/FNNDSC/ChRIS_ultron_backEnd/blob/master/chris_backend/config/settings/production.py) and [fnndsc/ChRIS_store](https://github.com/FNNDSC/ChRIS_store/blob/master/store_backend/config/settings/production.py).
3. Rebuild docker images.
4. Change [`docker-compose.yml`](https://github.com/FNNDSC/ChRIS_ultron_backEnd/blob/master/docker-compose.yml) to make sure your new images are being used.
5. Re-run.

# Examples

## Caddy

_Caddy_ sets up HTTPS automatically with _Let's Encrypt_ and its configuration syntax is easy to understand.

https://caddyserver.com/

The example below demonstrates how to expose a single-machine instance running [`./docker-deploy.sh up`](https://github.com/FNNDSC/ChRIS_ultron_backEnd/blob/master/docker-deploy.sh). CUBE and store "backends" are served by reverse-proxy. Logging is enabled for the two backends. _ChRIS_ui_ and _ChRIS_store_ui_ are served from a directory "on the metal." Both are react.js "single-page applications" (SPA), so every URI should return `/index.html`.

```caddyfile
cube.chris.example.com {
  reverse_proxy localhost:8000 {
    header_up X-Forwarded-Proto {http.request.scheme}
  }
  log
}

api.store.chris.example.com {
  reverse_proxy localhost:8010 {
    header_up X-Forwarded-Proto {http.request.scheme}
  }
  log
}

chris.example.com {
  root * /srv/ChRIS_ui/build
  file_server
  try_files {path} /index.html
}

store.chris.example.com {
  root * /srv/ChRIS_store_ui/build
  file_server
  try_files {path} /index.html
}

# optional but very awesome monitoring app
# https://github.com/netdata/netdata
netdata.chris.example.com {
  reverse_proxy localhost:19999
}
```

## Apache

You have:

- One server, one IP address, one domain name
- Backend is listening on `http://127.0.0.1:8010/api/v1/`
- [ChRIS_store_ui](https://github.com/FNNDSC/ChRIS_store_ui) cloned to `/var/src/ChRIS_store_ui`
- Domain name `chrisstore.co` has A record pointing to your server

You want:

- ChRIS_store_ui served on https://chrisstore.co/
- ChRIS_store (backend) served on https://chrisstore.co/api/v1/

### Reverse-proxy

```
ProxyPass "/api" "http://localhost:8010/api"
ProxyAddHeaders On  # this is the default anyways
RequestHeader set "X-Forwarded-Proto" expr=%{REQUEST_SCHEME}
```

You might need to first run `sudo a2enmod headers`

### Front-End SPA On-The-Metal

```
DocumentRoot /var/src/ChRIS_store_ui/build
```

Use `/var/src/ChRIS_store_ui/build/.htaccess` for SPA-support (single-page application).
This code serves `/index.html` for everything (not found on the filesystem in `DocumentRoot`).

```
Options -MultiViews
    RewriteEngine On
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^ index.html [QSA,L]
```

Alternatively, you can also run a static server in a container and use a reverse-proxy to serve the front-end too.

## Putting It All Together

### Basic Example

Create the file `/etc/apache2/sites-available/100-chrisstore.conf`

```
<VirtualHost *:80>
  ServerName chrisstore.co
  ProxyPass "/api" "http://localhost:8010/api"
  ProxyAddHeaders On  # this is the default anyways
  RequestHeader set "X-Forwarded-Proto" expr=%{REQUEST_SCHEME}
  DocumentRoot /var/src/ChRIS_store_ui/build

  ErrorLog ${APACHE_LOG_DIR}/chrisstore.error.log
  CustomLog ${APACHE_LOG_DIR}/chrisstore.access.log combined
</VirtualHost>
```

Run

```bash
sudo ln -s ../sites-enabled/100-chrisstore.conf /etc/apache2/sites-available/100-chrisstore.conf
sudo systemctl restart apache2
```

### Let's Encrypt + Certbot

```bash
# install (on Ubuntu)
sudo apt install certbot python3-certbot-apache
# enable HTTPS using magic. gets certs and write apache configs
sudo certbot --apache
```

### Full Vanilla Example

```
<VirtualHost *:80>
  ServerName chrisstore.co
  RewriteEngine on
  RewriteCond %{SERVER_NAME} =chrisstore.co
  RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>

<VirtualHost *:443>
  ServerName chrisstore.co

  ProxyPass "/api" "http://localhost:8010/api"
  ProxyAddHeaders On  # this is the default anyways
  RequestHeader set "X-Forwarded-Proto" expr=%{REQUEST_SCHEME}

  DocumentRoot /var/www/ChRIS_store_ui/build

  ErrorLog ${APACHE_LOG_DIR}/chrisstore.error.log
  CustomLog ${APACHE_LOG_DIR}/chrisstore.access.log combined

  SSLEngine on
  SSLCertificateFile /etc/letsencrypt/live/chrisstore.co/fullchain.pem
  SSLCertificateKeyFile /etc/letsencrypt/live/chrisstore.co/privkey.pem
  SSLProtocol             all -SSLv2 -SSLv3
  SSLCipherSuite          ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS
  SSLHonorCipherOrder     on
  SSLCompression          off
  SSLOptions +StrictRequire
</VirtualHost>
```