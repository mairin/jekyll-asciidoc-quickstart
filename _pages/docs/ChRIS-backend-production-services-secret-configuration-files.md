## Abstract

This page describes the configuration files required by the single-machine production deployment of the ChRIS backend services. Those files can contain secret variables such as API keys and authentication passwords as well as other configuration variables.

## Currently required files
* ``.chris.env``
* ``.chris_db.env``
* ``.chris_store.env``
* ``.chris_store_db.env``
* ``.swift_service.env``

Those files should be copied within a ``secrets`` folder created inside the source of the repo, like:

```bash
git clone https://github.com/FNNDSC/ChRIS_ultron_backend
cd ChRIS_ultron_backend
mkdir secrets
```


## Secret configuration file options

### ``.chris.env`` 

```bash
DJANGO_SETTINGS_MODULE=config.settings.production
DJANGO_ALLOWED_HOSTS=*
DJANGO_SECRET_KEY="key1"
DJANGO_CORS_ORIGIN_ALLOW_ALL=true
DJANGO_CORS_ORIGIN_WHITELIST=babymri.org
DJANGO_SECURE_PROXY_SSL_HEADER=
DJANGO_USE_X_FORWARDED_HOST=false
STATIC_ROOT=/home/localuser/mod_wsgi-0.0.0.0:8000/htdocs/static/
DATABASE_HOST=chris_db
DATABASE_PORT=3306
CHRIS_STORE_URL=http://chris_store:8010/api/v1/
SWIFT_CONTAINER_NAME=users
SWIFT_AUTH_URL=http://swift_service:8080/auth/v1.0
CELERY_BROKER_URL=amqp://queue:5672
PFCON_URL=http://pfcon_service:5005
```

### ``.chris_db.env``

```bash
MYSQL_ROOT_PASSWORD=password1
MYSQL_DATABASE=chris
MYSQL_USER=chris
MYSQL_PASSWORD=password2
```

### ``.chris_store.env``

```bash
DJANGO_SETTINGS_MODULE=config.settings.production
DJANGO_ALLOWED_HOSTS=*
DJANGO_SECRET_KEY="key2"
DJANGO_CORS_ORIGIN_ALLOW_ALL=true
DJANGO_CORS_ORIGIN_WHITELIST=babymri.org
DJANGO_SECURE_PROXY_SSL_HEADER=
DJANGO_USE_X_FORWARDED_HOST=false
DATABASE_HOST=chris_store_db
DATABASE_PORT=3306
SWIFT_AUTH_URL=http://swift_service:8080/auth/v1.0
SWIFT_CONTAINER_NAME=store_users
```

### ``.chris_store_db.env``

```bash
MYSQL_ROOT_PASSWORD=password3
MYSQL_DATABASE=chris_store
MYSQL_USER=chris
MYSQL_PASSWORD=password4
```

### ``.swift_service.env``

```bash
SWIFT_USERNAME=chris:password5
SWIFT_KEY=key3
```

## Reverse Proxy Settings

If the app is behind a reverse-proxy to enable HTTPS upgrade, in `.chris.env` and `.chris_store.env` set

```
DJANGO_SECURE_PROXY_SSL_HEADER=HTTP_X_FORWARDED_PROTO,https
DJANGO_USE_X_FORWARDED_HOST=true
```

See https://github.com/FNNDSC/ChRIS_ultron_backEnd/wiki/Deployment#fix

## Automatic Script

If you're using `./docker-deploy.sh` and want things to "just work," use this script to set random values to all the required variables.

```bash
#!/bin/bash
# purpose: set up secrets/*.env
# https://github.com/FNNDSC/ChRIS_ultron_backEnd/wiki/ChRIS-backend-production-services-secret-configuration-files

DJANGO_CORS_ORIGIN_ALLOW_ALL=${DJANGO_CORS_ORIGIN_ALLOW_ALL:-true}
DJANGO_CORS_ORIGIN_WHITELIST=${DJANGO_CORS_ORIGIN_WHITELIST:-"babymri.org"}

# Create a random mixed-case alphanumieric string of given length (default 60)
function generate_password () {
  head /dev/urandom | tr -dc A-Za-z0-9 | head -c "${1:-60}"
}

source_dir=$(dirname "$(readlink -f "$0")")
secrets_dir=$source_dir/secrets

if [ -d "$secrets_dir" ]; then
  echo $secrets_dir already exists
  exit 1
fi

mkdir $secrets_dir
cd $secrets_dir

cat > .chris.env << EOF
DJANGO_SETTINGS_MODULE=config.settings.production
DJANGO_ALLOWED_HOSTS=*
DJANGO_SECRET_KEY=$(generate_password)
DJANGO_CORS_ORIGIN_ALLOW_ALL=$DJANGO_CORS_ORIGIN_ALLOW_ALL
DJANGO_CORS_ORIGIN_WHITELIST=$DJANGO_CORS_ORIGIN_WHITELIST
STATIC_ROOT=/home/localuser/mod_wsgi-0.0.0.0:8000/htdocs/static/
DJANGO_SECURE_PROXY_SSL_HEADER=
DJANGO_USE_X_FORWARDED_HOST=false
DATABASE_HOST=chris_db
DATABASE_PORT=3306
CHRIS_STORE_URL=http://chris-store.local:8010/api/v1/
SWIFT_CONTAINER_NAME=users
SWIFT_AUTH_URL=http://swift_service:8080/auth/v1.0
CELERY_BROKER_URL=amqp://queue:5672
EOF

cat > .chris_db.env << EOF
MYSQL_ROOT_PASSWORD=$(generate_password)
MYSQL_DATABASE=chris
MYSQL_USER=chris
MYSQL_PASSWORD=$(generate_password)
EOF

cat > .chris_store.env << EOF
DJANGO_SETTINGS_MODULE=config.settings.production
DJANGO_ALLOWED_HOSTS=*
DJANGO_SECRET_KEY=$(generate_password)
DJANGO_CORS_ORIGIN_ALLOW_ALL=$DJANGO_CORS_ORIGIN_ALLOW_ALL
DJANGO_CORS_ORIGIN_WHITELIST=$DJANGO_CORS_ORIGIN_WHITELIST
DATABASE_HOST=chris_store_db
DATABASE_PORT=3306
SWIFT_AUTH_URL=http://swift_service:8080/auth/v1.0
SWIFT_CONTAINER_NAME=store_users
DJANGO_SECURE_PROXY_SSL_HEADER=
DJANGO_USE_X_FORWARDED_HOST=false
EOF

cat > .chris_store_db.env << EOF
MYSQL_ROOT_PASSWORD=$(generate_password)
MYSQL_DATABASE=chris_store
MYSQL_USER=chris
MYSQL_PASSWORD=$(generate_password)
EOF

# this is hard coded
cat > .swift_service.env << EOF
SWIFT_USERNAME=chris:chris1234
SWIFT_KEY=testing
EOF

# wrapper around generate_password to print a newline after the result
function print_password () {
  generate_password $1
  printf "\n"
}

echo "Here are some more passwords for you to use for when setting up superuser accounts"
print_password 8
print_password 8
print_password 8
print_password 8
print_password 12
print_password 12
print_password 12
print_password 12
print_password 60
print_password 60
print_password 60
print_password 60
```