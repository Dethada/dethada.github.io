---
layout: post
header-style: text
title:  "Firefly III Setup with Docker and HTTPS"
date:   2019-04-06 00:00:00
author: "David"
tags:
    - Tutorial
    - Nginx
    - Docker
---

In this tutorial we will setup [Firefly III](https://firefly-iii.org/) using docker and setup a reverse proxy to enable https, as *Firefly III* itself does not support https. For the purpose of this tutorial we will be using `firefly.example.com` as the domain.

> Note: This tutorial assumes you have already setup a mysql/postgres database.

## Docker
> If you have not yet installed docker refere to [docker install documentation](https://docs.docker.com/install/) to install it first.

First we create persistent volumes to store uploaded files and exported data.
```bash
docker volume create firefly_iii_export
docker volume create firefly_iii_upload
```

To ensure the site works behind our reverse proxy and all the links on the site is using https we have to set the following environment variables.
```bash
APP_URL=https://firefly.example.com
TRUSTED_PROXIES=**
```

The app key is any 32 character alphanumeric string. The database by default is assumed to be MySQL, if you are using a Postgres database you have to set an extra environment vairable `DB_CONNECTION=pgsql`.

The following command will run the firefly-iii container and map it to port `4040` on the host.
```bash
docker run -d \
-v firefly_iii_export:/var/www/firefly-iii/storage/export \
-v firefly_iii_upload:/var/www/firefly-iii/storage/upload \
-p 127.0.0.1:4040:80 \
-e FF_APP_ENV=local \
-e FF_APP_KEY=12345678901234567890123456789012 \
-e FF_DB_HOST=CHANGEME \
-e FF_DB_PORT=CHANGEME \
-e FF_DB_NAME=CHANGEME \
-e FF_DB_USER=CHANGEME \
-e FF_DB_PASSWORD=CHANGEME \
-e APP_URL=https://firefly.example.com \
-e TRUSTED_PROXIES=** \
--name firefly-iii-c1 \
jc5x/firefly-iii:latest
```

## Enable Recurring Transactions
> You can ignore this if you are not planning on using recurring transactions.

Drop into shell on the container
```bash
docker exec -it firefly-iii-c1 /bin/bash
```

Install cron in docker container
```bash
apt update
apt install cron
```

Run `crontab -e` to edit cronjobs, then add the following cron job to enable recurring transactions.
```
0 0 * * * /usr/local/bin/php /var/www/firefly-iii/artisan firefly:cron
```

These changes will persist even if you restart the container, however if you start another container from the image `jc5x/firefly-iii:latest` you have to do these steps again.

## Setup Nginx
If you are using cloudflare as your dns provider, you can refer to [this post]({% post_url 2019-04-03-Certbot-Cloudflare-DNS-Plugin %}) on getting TLS certificates from *Let's Encrypt* using the cloudflare dns plugin.

This is a sample site configuration for nginx. Change `firefly.example.com` to your domain. You should also change the `proxy_pass` parameter on line 17 if you mapped the host port of *Firefly III* to a port other than `4040`, or if the docker container is running on another host.
```
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name firefly.example.com;

    # SSL
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    # logging
    access_log /var/log/nginx/firefly.example.com.access.log;
    error_log /var/log/nginx/firefly.example.com.error.log warn;

    # reverse proxy
    location / {
        proxy_pass http://127.0.0.1:4040;
        proxy_http_version      1.1;
        proxy_cache_bypass      $http_upgrade;
        proxy_set_header Upgrade                $http_upgrade;
        proxy_set_header Connection             "upgrade";
        proxy_set_header Host                   $host;
        proxy_set_header X-Real-IP              $remote_addr;
        proxy_set_header X-Forwarded-For        $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto      $scheme;
        proxy_set_header X-Forwarded-Host       $host;
        proxy_set_header X-Forwarded-Port       $server_port;
    }
    
    # security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src * data: 'unsafe-eval' 'unsafe-inline'" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # . files
    location ~ /\.(?!well-known) {
        deny all;
    }

    # gzip
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript application/xml+rss applica
    tion/atom+xml image/svg+xml;
}

# subdomains redirect
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name *.firefly.example.com;

    # SSL
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    return 301 https://firefly.example.com$request_uri;
}

# HTTP redirect
server {
    listen 80;
    listen [::]:80;
    server_name .firefly.example.com;

    location / {
        return 301 https://firefly.example.com$request_uri;
    }
}
```

## Resources
- [Installation Documentation](https://docs.firefly-iii.org/en/latest/installation/docker.html)
- [Cronjob Documentation](https://docs.firefly-iii.org/en/latest/installation/cronjob.html#cronjobs)