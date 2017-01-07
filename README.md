# letsencrypt_nginx

Documenting setting up Nginx with Let's Encrypt SSL on a Arch Linux VPS with reverse proxy and subdomains

Install certbot and nginx
```
sudo pacman -S nginx certbot
```

Start and enable Nginx
```
sudo systemctl enable nginx
sudo systemctl start nginx
```

Set base Nginx config (from Arch Wiki): /etc/nginx/nginx.conf
```
user http;
worker_processes auto;
worker_cpu_affinity auto;
pcre_jit on;

events {
    worker_connections 2048;
}


http {
    include mime.types;
    default_type application/octet-stream;
    sendfile on;
    tcp_nopush on;
    aio threads;
    server_tokens off; # Security: Disables nginx version in error messages and in the “Server” response header field.
    charset utf-8; # Force usage of UTF-8
    index index.php index.html index.htm;
    include servers-enabled/*; # See Server blocks
}
```

I like to organize sites in separate directories like Apache Virtual Hosts
```
mkdir /etc/nginx/sites-available
mkdir /etc/nginx/sites-enabled
```

Edit permission on /var/lib/letsencrypt/ so http user can write challenge files needed for Let's Encrypt webroot verification
```
chgrp http /var/lib/letsencrypt
chmod g+s /var/lib/letsencrypt
```

Create a letsencrypt.conf file: /etc/nginx/letsencrypt.conf
```
location ^~ /.well-known/acme-challenge {
  alias /var/lib/letsencrypt/.well-known/acme-challenge;
  default_type "text/plain";
  try_files $uri =404;
}
```

Create site config file. In this case here's a gogs git server so: /etc/nginx/sites-available/gogs
```
# redirect to https

server {
  listen 80;
  listen [::]:80;
  server_name git.mydomain.com;
  return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    ssl_certificate /etc/letsencrypt/live/mydomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mydomain.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/mydomain.com/chain.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;
    ssl_protocols TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
    ssl_prefer_server_ciphers on;
    add_header Strict-Transport-Security max-age=15768000;
    ssl_stapling on;
    ssl_stapling_verify on;

    server_name git.mydomain.com;
    include letsencrypt.conf;

    location / {
        proxy_pass http://localhost:4444;
    }
}
```

To enable the server, create a symlink
```
ln -s /etc/nginx/sites-available/gogs /etc/nginx/sites-enabled/gogs
```

Add all your other individual site config as separate files and enable those with symlinks.

Grab a cert from Let's Encrypt. Add all your subdomains.
```
sudo certbot certonly --webroot -w /var/lib/letsencrypt/ -d mydomain.com -d git.mydomain.com -d www.mydomain.com
```

If verification checked out you should have a positive message like this
```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/mydomain.com/fullchain.pem. Your cert
   will expire on 2017-04-07. To obtain a new or tweaked version of
   this certificate in the future, simply run certbot again. To
   non-interactively renew *all* of your certificates, run "certbot
   renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```


Check for Nginx config errors
```
sudo nginx -t
```

Restart Nginx
```
sudo systemctl restart nginx
```

Check journal if errors
```
sudo journalctl -xe
```

At this point, I edited my DNS settings.
```
A Record mydomain.com -> 123.456.789.111
CNAME git.mydomain.com -> mydomain.com
```

Check name resolution.
```
nslookup mydomain.com
nslookup git.mydomain.com
```

Test renewal
```
certbot renew --dry-run
```

To auto renew:
Create /etc/systemd/system/certbot.service
```
[Unit]
Description=Let's Encrypt renewal

[Service]
Type=oneshot
#ExecStart=/usr/bin/certbot renew --dry-run # A trial run to verify things are working
ExecStart=/usr/bin/certbot renew --quiet --agree-tos
ExecStartPost=/bin/systemctl reload nginx.service

[Install]
WantedBy=multi-user.target
```

Create /etc/systemd/system/certbot.timer
```
[Unit]
Description=Daily renewal of Let's Encrypt's certificates

[Timer]
OnCalendar=daily
RandomizedDelaySec=1day
Persistent=true

[Install]
WantedBy=timers.target
```

Enable and start certbot.timer
```
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer
```

Check journal for certbot logs
```
sudo journalctl -u certbot.service
```

Further Reading:
* https://certbot.eff.org/#arch-nginx
* https://wiki.archlinux.org/index.php/Let%E2%80%99s_Encrypt
* https://wiki.archlinux.org/index.php/nginx
