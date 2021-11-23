---
title:  "Certbot - Cloudflare DNS Plugin"
date:   2019-04-02 00:00:00
author: "David"
tags:
  - Tutorial
  - DNS
---

In this tutorial we will get a wildcard certificate from letsencrypt using the cloudflare dns plugin. For the purpose of this tutorial we will be using `example.com` as the domain.

### Install Cloudflare DNS Plugin
This tutorial assumes you have already installed certbot. If you have not, you can follow the instructions from [certbot-eff][certbot-eff].
```bash
sudo apt update
sudo apt install python3-certbot-dns-cloudflare -y
```

### API Credentials
```bash
mkdir -p /root/secrets/certbot
```

Retrieve your api key from cloudflare.
> 1. Login to the Cloudflare account.
> 2. Go to My Profile.
> 3. Scroll down to API Keys and locate Global API Key.
> 4. Click API Key to see your API identifier.

Create the file below with your cloudflare information. We will save the file at `/root/secrets/certbot/cloudflare.ini`.
```
# Cloudflare API credentials used by Certbot
dns_cloudflare_email = cloudflare@example.com
dns_cloudflare_api_key = 0123456789abcdef0123456789abcdef01234567
```

Secure the folder and file.
```bash
find /root/secrets -type d -exec chmod 700 {} \;
find /root/secrets -type f -exec chmod 600 {} \;
```

### Requesting for Certificate
The `--dns-cloudflare-propagation-seconds` option defines the number of seconds to wait before doing the validation checks, you can change it accordingly.

It is important that we specify the server to be the ACME v2 server as the v1 server does not support wildcard certificates.
```bash
certbot certonly \
  --preferred-challenges dns \
  --email admin@example.com
  --dns-cloudflare-credentials /root/secrets/certbot/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d "*.example.com" -d example.com \
  --server https://acme-v02.api.letsencrypt.org/directory \
  --agree-tos
```

We should now have our cert at `/etc/letsencrypt/live/example.com/`.
```bash
root@server:~# ls -la /etc/letsencrypt/live/example.com
total 12
drwxr-xr-x 2 root root 4096 Apr  3 10:44 .
drwx------ 3 root root 4096 Apr  3 10:44 ..
lrwxrwxrwx 1 root root   34 Apr  3 10:44 cert.pem -> ../../archive/example.com/cert1.pem
lrwxrwxrwx 1 root root   35 Apr  3 10:44 chain.pem -> ../../archive/example.com/chain1.pem
lrwxrwxrwx 1 root root   39 Apr  3 10:44 fullchain.pem -> ../../archive/example.com/fullchain1.pem
lrwxrwxrwx 1 root root   37 Apr  3 10:44 privkey.pem -> ../../archive/example.com/privkey1.pem
-rw-r--r-- 1 root root  692 Apr  3 10:44 README
```

## Resources
- [Certbot-eff][certbot-eff]
- [certbot-dns-cloudflare][dns-cloudflare]
- [Cloudflare API Key](https://support.cloudflare.com/hc/en-us/articles/200167836-Where-do-I-find-my-Cloudflare-API-key-)

[certbot-eff]: https://certbot.eff.org/
[dns-cloudflare]: https://certbot-dns-cloudflare.readthedocs.io/en/stable/
