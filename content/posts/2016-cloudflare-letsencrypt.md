---
title: "Cloudflare with LetsEncrypt SSL Certificates + Apache's ProxyPass"
date: 2016-11-08T14:53:00+00:00
---

So, you want to use [LetsEncrypt](https://letsencrypt.org/) to generate a free SSL certificate for your site behind Cloudflare and Apache (acting as a ReverseProxy). After many hours of research and tweaking, I got this setup to work. Here I will document my process for you to use. In the end, your setup will look something like this: _User <-(SSL)-> Cloudflare <-(SSL)-> Apache <--> Your Webapp (not SSL)_. This guide assumes you have a site already setup without SSL. I am using Ubuntu 16.04 for my setup, but the process should be similar for other OSes.

## Getting Started

Before we do anything, we need to install LetsEncrypt. LetsEncrypt reccomends [Certbot](https://certbot.eff.org/) as their client. Unless you know what you are doing, you should use this. Follow the instructions on their site to install the Certbot client. After Certbot is installed, you should install the apache mod *mod_cloudflare*. It isn't required, but is good in any setup invloving Cloudflare, as it resolves IPs to their actual source, not Cloudflare's IPs. Follow the instructions on [Cloudflare's site](https://www.cloudflare.com/technical-resources/#mod_cloudflare), and run `sudo a2enmod cloudflare` to ensure it is enabled. Finally, make a backup of the the site configs you will be changing. It is unlikely something will go wrong, but just in case we need to roll back we should have one.

## Custom Config Patch

Now that Certbot/LE is installed, we will continue on with the setup. By default, apache passes all traffic to the backend application when using ProxyPass. We need to add an exception to this rule by installing and including a custom config. *If you aren't using ProxyPass, skip this step*. Make a new file in `/etc/apache2/conf-avaliable/letsencrypt-proxypass-fix.conf`, and paste the following into it:

```apache
# From https://github.com/certbot/certbot/issues/2164
<IfModule mod_proxy.c> # If Proxy is installed
    ProxyPass "/.well-known/acme-challenge" "!" # Do not proxy the acme verification
</IfModule>
Alias /.well-known/acme-challenge /var/www/html/.well-known/acme-challenge # Send all requests for the acme verification to this location
<Directory "/var/www/html/.well-known/acme-challenge"> # Allow any requests for it
    Options None
    AllowOverride None
    Require all granted
    AddDefaultCharset off
</Directory>
```

Save the file, and quit. For now this isn't activated, but we will be including it in the site configuration file next.

## Generating the Certificate

Now that most of the prep work is finished, we can get to actually generating the certificate. Include the line `Include /etc/apache2/conf-available/letsencrypt-proxypass-fix.conf` in the site's config, and reload apache. Now, run `certbot certonly --webroot --webroot-path /var/www/html/ --renew-by-default -d YOUR.DOMAIN.HERE`. Answer the questions, and assuming everything was done correctly, you will get a message saying that the certificate was generated successfully, and will expire in 90 days. Whoopee, you have an SSL certificate! However, even though you were issued an SSL certificate, apache doesn't know about it, and is still only listening for non-secure connections. We need to install the certificate for apache to use.

## Installing the Certificate
Now that you have generated the certificate, you need to install it to apache. Open your site's config, and edit it to look something like the following. Note, some applications require extra flags and steps, so please follow their documentation as well; this is a minimal config. Also, don't forget to include any other configurations you had from your old config!

```apache
<VirtualHost *:80>
ServerName YOUR.DOMAIN.HERE
Redirect permanent / https://YOUR.DOMAIN.HERE/ # Redirect all nonsecure requests to use HTTPS
</VirtualHost>

<VirtualHost *:443>
ServerName YOUR.DOMAIN.HERE

SSLProxyEngine On
SSLCertificateFile /etc/letsencrypt/live/YOUR.DOMAIN.HERE/cert.pem
SSLCertificateKeyFile /etc/letsencrypt/live/YOUR.DOMAIN.HERE/privkey.pem
SSLCertificateChainFile /etc/letsencrypt/live/YOUR.DOMAIN.HERE/chain.pem
Include /etc/letsencrypt/options-ssl-apache.conf # The LetsEncrypt SSL config
Include /etc/apache2/conf-available/letsencrypt-proxypass-fix.conf # The patch for ProxyPass we made earlier

ProxyRequests Off
ProxyPreserveHost On
ProxyPass / http://000.000.000.000/ # Replace this with whatever address you had from your old config
ProxyPassReverse / http://000.000.000.000/

RequestHeader append "X-Forwarded-Proto" "https" # Tell the application we are forwarding from SSL
RequestHeader set "X-Forwarded-Ssl" "on"
</VirtualHost>
```

Tada! Your service now uses your certificate. If you were to visit this site without Cloudflare's CDN you would be able to see that the site uses LetsEncrypt.

## Configuring Cloudflare
Now that HTTPS is installed, we need to tell Cloudflare to use Full SSL. I personally have had mixed experiences, but sometimes if this isn't set properly, your browser can get stuck in a Redirect loop. Full SSL requires that your server has a certificate installed for users to access HTTPS via Cloudflare. Changing this option affects all records on the domain, so don't do this unless everything is set up correctly (alternativley you can use a [Page Rule](https://support.cloudflare.com/hc/en-us/articles/200170536-How-do-I-redirect-all-visitors-to-HTTPS-SSL-)). For a more detailed explanation, see [Cloudflare's support article](https://support.cloudflare.com/hc/en-us/articles/200170416-What-do-the-SSL-options-mean-).

Once you are logged into Cloudflare's website, click your domain, click the *Crypto* button, and select *Full (Strict)* from the dropdown. Now Cloudflare will only use your valid LetsEncrypt Certificate for communication!

## Wrap Up
If all went according to plan, you should be able to naivgate your site from inside or ourside Cloudflare via HTTPS. Make sure you update your site's links to use HTTPS! 
Don't forget every month or so to renew the certificate you just made with `certbot renew`. I will make another post later on how to automatically renew your certs, eliminating the need for maintenance.

I hope this guide helped you. If you have any questions or comments, don't be afraid to comment or contact me!

