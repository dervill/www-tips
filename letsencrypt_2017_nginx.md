# How to setup Let's Encrypt for Nginx on Ubuntu 16.04 (including IPv6, HTTP/2 and A+ SLL rating)

There are two main modes to run the Let's Encrypt client (called `Certbot`):
 - [Standalone](https://certbot.eff.org/docs/using.html#standalone): replaces the webserver to respond to ACME challenges
 - [Webroot](https://certbot.eff.org/docs/using.html#webroot): needs your webserver to serve challenges from a known folder.

**Webroot is better** because it doesn't need to replace Nginx (to bind to port 80).

In the following, we're setting up `mydomain.com`.
HTML is served from `/var/www/mydomain`, and challenges are served from `/var/www/letsencrypt`.


-------------------------------------------------------------------------------

## Nginx snippets

First we create two snippets (to avoid duplicating code in every virtual host configuration).

Create a file `/etc/nginx/snippets/letsencrypt.conf` containing:

	location ^~ /.well-known/acme-challenge/ {
		default_type "text/plain";
		root /var/www/letsencrypt;
	}


Create a file `/etc/nginx/snippets/ssl.conf` containing:

	ssl_session_timeout 1d;
	ssl_session_cache shared:SSL:50m;
	ssl_session_tickets off;

	ssl_protocols TLSv1.2;
	ssl_ciphers EECDH+AESGCM:EECDH+AES;
	ssl_ecdh_curve secp384r1;
	ssl_prefer_server_ciphers on;

	ssl_stapling on;
	ssl_stapling_verify on;

	add_header Strict-Transport-Security "max-age=15768000; includeSubdomains; preload";
	add_header X-Frame-Options DENY;
	add_header X-Content-Type-Options nosniff;

Create the folder for the challenges:

	sudo mkdir -p /var/www/letsencrypt/.well-known/acme-challenge


-------------------------------------------------------------------------------

## Nginx virtual hosts (HTTP-only)

We don't have a certificate yet at this point, so the domain will be served only as HTTP.

Create a file `/etc/nginx/sites-available/mydomain.conf` containing:

	server {
		listen 80 default_server;
		listen [::]:80 default_server ipv6only=on;
		server_name mydomain.com www.mydomain.com;

		include /etc/nginx/snippets/letsencrypt.conf;

		root /var/www/mydomain;
		index index.html;
		location / {
			try_files $uri $uri/ =404;
		}
	}

Enable the site:

	rm /etc/nginx/sites-enabled/default
	ln -s /etc/nginx/sites-available/mydomain.conf /etc/nginx/sites-enabled/mydomain.conf

And reload Nginx:

	sudo service nginx reload


-------------------------------------------------------------------------------

## Certbot

Install the package:

	sudo apt-get install software-properties-common
	sudo add-apt-repository ppa:certbot/certbot
	sudo apt-get update
	sudo apt-get install certbot

Note: there is also a `letsencrypt` package in APT, but it's a much older version of the client.


-------------------------------------------------------------------------------

## Get the certificate

Request the certificate (don't forget to replace with your own email address):

	certbot certonly --webroot --agree-tos --no-eff-email --email YOUR@EMAIL.COM -w /var/www/letsencrypt -d www.domain.com -d domain.com

It will save the files in `/etc/letsencrypt/live/www.mydomain.com/`.

Note: The flag `--no-eff-email` opts out of signing up for the [EFF mailing list](https://lists.eff.org/cgi-bin/mailman/listinfo), remove the flag if you'd like to signup.


----

## Nginx virtual hosts (HTTPS-only)

Now that you have a certificate for the domain, switch to HTTPS by editing the file `/etc/nginx/sites-available/mydomain.conf` and replacing contents with:

	## http://mydomain.com redirects to https://mydomain.com
	server {
		listen 80;
		listen [::]:80;
		server_name mydomain.com;

		include /etc/nginx/snippets/letsencrypt.conf;

		location / {
			return 301 https://mydomain.com$request_uri;
		}
	}

	## http://www.mydomain.com redirects to https://www.mydomain.com
	server {
		listen 80 default_server;
		listen [::]:80 default_server ipv6only=on;
		server_name www.mydomain.com;

		include /etc/nginx/snippets/letsencrypt.conf;

		location / {
			return 301 https://www.mydomain.com$request_uri;
		}
	}

	## https://mydomain.com redirects to https://www.mydomain.com
	server {
		listen 443 ssl http2;
		listen [::]:443 ssl http2;
		server_name mydomain.com;

		ssl_certificate /etc/letsencrypt/live/www.mydomain.com/fullchain.pem;
		ssl_certificate_key /etc/letsencrypt/live/www.mydomain.com/privkey.pem;
		ssl_trusted_certificate /etc/letsencrypt/live/www.mydomain.com/fullchain.pem;
		include /etc/nginx/snippets/ssl.conf;

		location / {
			return 301 https://www.mydomain.com$request_uri;
		}
	}

	## Serves https://www.mydomain.com
	server {
		server_name www.mydomain.com;
		listen 443 ssl http2 default_server;
		listen [::]:443 ssl http2 default_server ipv6only=on;

		ssl_certificate /etc/letsencrypt/live/www.mydomain.com/fullchain.pem;
		ssl_certificate_key /etc/letsencrypt/live/www.mydomain.com/privkey.pem;
		ssl_trusted_certificate /etc/letsencrypt/live/www.mydomain.com/fullchain.pem;
		include /etc/nginx/snippets/ssl.conf;

		root /var/www/mydomain;
		index index.html;
		location / {
			try_files $uri $uri/ =404;
		}
	}


Then reload Nginx:

	sudo service nginx reload

Note that `http://mydomain.com` redirects to `https://mydomain.com` (which redirects to `https://www.mydomain.com`)
because redirecting to `https://www.mydomain.com` directly would be incompatible with HSTS.


---

## Automatic renewal using Cron

Certbot can renew all certificates that expire within 30 days, so let's make a cron for it.
You can test it has the right config by launching a dry run:

	certbot renew --dry-run

Create a file `/root/letsencrypt.sh`:

	#!/bin/bash
	systemctl reload nginx

	# If you have other services that use the certificates:
	# systemctl restart mosquitto

Make it executable:

	chmod +x /root/letsencrypt.sh

Edit cron:

	sudo crontab -e

And add the line:

	20 3 * * * certbot renew --noninteractive --renew-hook /root/letsencrypt.sh


----

## Conclusion

Congratulations, you should now be able to see your website at `https://www.mydomain.com` 🙂

You can now also test that your domain has A+ SLL rating:
 - https://www.ssllabs.com/ssltest/analyze.html?d=mydomain.com
 - https://www.ssllabs.com/ssltest/analyze.html?d=www.mydomain.com

I would also recommend setting up content-specific features like `Content Security Policy` and `Subresource Integrity`:
 - [Mozilla Observatory](https://observatory.mozilla.org): submit a domain to get content-specific advices
 - [Mozilla Security Guidelines](https://wiki.mozilla.org/Security/Guidelines/Web_Security)

If Let's Encrypt is useful to you, consider [donating to Let's Encrypt](https://letsencrypt.org/donate/) or [donating to the EFF](https://supporters.eff.org/donate/).
