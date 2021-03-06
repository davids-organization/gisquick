Application is split into 3 services running in docker containers:

#### QGIS Server
* Image: `gisquick/qgis-server`
* Volumes:
  - `/publish/` - for published Gisquick projects

#### Django Application (served with Gunicorn)
* Image: `gisquick/django`
* Volumes:
  - `/var/www/data/` - for sqlite database
  - `/var/www/media/` - for tilecache

Build:
```
$ docker build -f docker/django/Dockerfile -t gisquick/django:1.0 --build-arg version=1.0 .
```

#### Nginx Server
* Image: `gisquick/nginx`
* Volumes:
  - `/etc/letsencrypt/` - it must contain ssl certificates before starting container (fullchain.pem and privkey.pem in live/projects.gisquick.org/ folder)
  - `/var/www/certbot/` - when you intend to use *Webroot mode* to generate new or renew existing Certbot's ssl certificates


## SSL certificates

### Self-Signed certificate


```
$ openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
    -keyout privkey.pem \
    -out fullchain.pem \
    -subj "/C=CZ/ST=Prague/L=Prague/O=Gisquick/OU=IT Department/CN=projects.gisquick.org"
```


### Certbot (LetsEncrypt) certificate


Create empty folders for data volumes mounted by nginx container
```
$ mkdir -p /var/www/certbot
$ mkdir -p /etc/letsencrypt
```


Create certificate without running containers - using Standalone mode
```
$ docker run --rm -it \
    -v "/etc/letsencrypt/:/etc/letsencrypt/" \
    -p 80:80 -p 443:443 \
    certbot/certbot certonly --standalone --agree-tos \
    --email admin@gisquick.org \
    -d projects.gisquick.org
```


Create certificate with running containers - using Webroot mode
```
$ docker run --rm -it \
    -v "/var/www/certbot/:/var/www/certbot/" \
    -v "/etc/letsencrypt/:/etc/letsencrypt/" \
    certbot/certbot certonly --agree-tos \
    -a webroot --webroot-path /var/www/certbot/ \
    --email admin@gisquick.org \
    -d projects.gisquick.org
```


Renew certificate
```
$ docker run --rm -it \
    -v "/var/www/certbot/:/var/www/certbot/" \
    -v "/etc/letsencrypt/:/etc/letsencrypt/" \
    certbot/certbot renew --quiet
```


Reload NGINX server
```
$ docker kill -s HUP `docker ps -qf "ancestor=gisquick/nginx"`
```



## Create regular users

Start Django admin-shell

a) use running django container
```
$ docker exec -it `docker ps -qf "ancestor=gisquick/django"` django-admin shell
```
b) or start in a new container
```
$ docker run -it --rm -v "/var/www/data/:/var/www/gisquick/data/" gisquick/django django-admin shell
```

Then you can create users programmatically
```python
from django.contrib.auth import get_user_model
get_user_model().objects.create_user('user', email='user@gisquick.org', password='user', first_name='User')
```


## Web client development

Setup folders for published projects, data and media files and create user with username 'user1' and password 'user1'.
Then start docker containers:

```
$ docker build -t gisquick/js-dev:alpine -f docker/client/alpine/Dockerfile .
$ docker-compose -f docker/docker-compose-dev-client.yml up
```

Open http://localhost:8100
