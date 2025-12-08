---
title: "How to obtain free SSL certificates with Letsencrypt, Certbot, and Cloudflare"
date: 2021-06-15T23:30:00-05:00
# weight: 1
# aliases: ["/first"]
tags: ["ssl", "nginx", "letsencrypt"]
categories: ["Selfhosting"]
#author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Get valid certificates for your domain"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
#editPost:
#    URL: "https://github.com/jjp17.github.io/content"
#    Text: "Suggest Changes" # edit text
#    appendFilePath: true # to append file path to Edit link
---

## Abstract
Let's say  you host some services by yourself. You should make your services secure right?  
Maybe you want your users to feel safe while using your apps? Maybe you just want to see a little green padlock next to your URLs?  
Either way, this guide will show you step by step how to obtain free SSL certs for your webserver using Letsencrypt, Certbot, and Cloudflare.

***

## Why
Using this method, you can have valid ssl certs to use on your https websites or apps.  
They will be signed by `letsencrypt` and should be auto renewed with `certbot`  
No ports are required to be opened for this since it uses api to modify DNS records. This allows for valid ssl certs to be used by any site, including internal only ones.

  
## Prerequisites
This guide assumes you have a few things set up already.

* A running webserver of any kind
* Root or Sudo access to the webserver
* A Cloudflare account
* A domain name
* Your domains DNS records in Cloudflare

  
## Installing
We will follow this guide to install certbot with snap (for some reason...) Choose the correct webserver and distro. All steps are outlined below. You must follow these steps on the machine with your web server.  
[Installing Certbot, for NGINX on Debian 10](https://certbot.eff.org/lets-encrypt/debianbuster-nginx)

### Install Snap

Install snap if not installed already. If snap is installed already, you can skip this section. These instructions are for Debian based distros.  

Update apt
```
sudo apt update
sudo apt install snapd
```

Install Core
```
sudo snap install core
```

Test the install
```
sudo snap install hello-world
hello-world
```


### Install Certbot

Since we need to use the snap version, make sure there are no other versions of certbot installed  
```
sudo apt remove certbot
```

Install certbot via snap
```
sudo snap install --classic certbot
```

Execute the following instruction on the command line on the machine to ensure that the certbot command can be run. 
```
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

Confirm plugin containment level  
Run this command on the command line on the machine to acknowledge that the installed plugin will have the same classic containment as the Certbot snap.
```
 sudo snap set certbot trust-plugin-with-root=ok
```

Install the correct plugin for dns challenges. We will use cloudflares
```
sudo snap install certbot-dns-cloudflare
```

  
## Setting up Cloudflare credentials
The cloudflare dns challenge plugin needs credentials to be able to insert a TXT DNS record via cloudflare API.  


### Generate a Cloudflare token
You need a token from cloudflare to be able to access their api. From the plugin page:

> Use of this plugin requires a configuration file containing Cloudflare API credentials, obtained from your Cloudflare dashboard.  
> Previously, Cloudflare’s “Global API Key” was used for authentication, however this key can access the entire Cloudflare API for all domains in your account, meaning it could cause a lot of damage if leaked.  
> Cloudflare’s newer API Tokens can be restricted to specific domains and operations, and are therefore now the recommended authentication option.  
> The Token needed by Certbot requires Zone:DNS:Edit permissions for only the zones you need certificates for.  
> Using Cloudflare Tokens also requires at least version 2.3.1 of the cloudflare python module. If the version that automatically installed with this plugin is older than that, and you can’t upgrade it on your system, you’ll have to stick to the Global key.  

1. Go to your [Cloudflare Dashboard](https://dash.cloudflare.com)  
2. Navigate to My Profile -> API Tokens -> Create Token  
3. Use the `Edit Zone DNS` template
4. Edit the details to your liking. Just be sure that DNS Zone can be edited for the correct domain
5. Continue to summary -> Create token
6. Copy your token to paste it into a config file later. The token will not be shown again if you navigate away from the page.

### Create the config file
Find a suitable place to hold the token config file and use an editor to make a file to paste it in. You could use roots home folder.
```
sudo nano /root/cloudflare.ini
```

Paste the token in. Use this template
```
# Cloudflare API token used by Certbot
dns_cloudflare_api_token = 0123456789abcdef0123456789abcdef01234567
```

If you are unable to use a token and must use the Global API Key, use this
```
# Cloudflare API credentials used by Certbot
dns_cloudflare_email = cloudflare@example.com
dns_cloudflare_api_key = 0123456789abcdef0123456789abcdef01234
```

  
## Run Certbot
Now that certbot, the Cloudflare DNS plugin, and the config file is in place, we run certbot to get an ssl cert

To acquire a certificate for example.com
```
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/cloudflare.ini \
  -d example.com
```

To acquire a single certificate for both example.com and www.example.com
```
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/cloudflare.ini \
  -d example.com \
  -d www.example.com
```

To acquire a wildcard certificate for example.com
```
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/cloudflare.ini \
  -d "*.example.com"
```

To acquire a certificate for example.com, waiting 60 seconds for DNS propagation
```
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d example.com
```

Once you have ran a command from above, test that certbot will automatically renew your cert.
```
sudo certbot renew --dry-run
```

  
## Add the certificate to NGINX
We must use the new certificate on our web-server. This is using NGINX
  
### Make an include file (Optional)
It would be easiest to make an includes file so we don't need to retype things all the time.
  
Make a directory to hold our include files
```
sudo mkdir -p /etc/nginx/includes
```

Make a config file for our ssl
```
sudo nano /etc/nginx/includes/ssl-cert.conf
```

Add the following lines into it. This assumes you had a wildcard cert.  
Change <domain> to whatever you got a cert for in the last section.
```
ssl_certificate      /etc/letsencrypt/live/<domain>/fullchain.pem;
ssl_certificate_key  /etc/letsencrypt/live/<domain>/privkey.pem;
```


### Include your cert in NGINX servers
Now that we have our cert, we include it in any virtual servers in NGINX

#### With includes file
In your virtual server, use the include directive to use the conf file we created above. Add this to your server:
```
include /etc/nginx/includes/ssl-cert.conf;
```

#### Without includes file
If you didn't make an includes file, just be sure to include the right NGINX commands. Be sure to change `<domain>` to the domain you got a certificate for.
```
ssl_certificate      /etc/letsencrypt/live/<domain>/fullchain.pem;
ssl_certificate_key  /etc/letsencrypt/live/<domain>/privkey.pem;
```