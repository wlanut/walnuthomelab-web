---
title: "SWAG Reverse Proxy, handling multiple domains across multiple servers"
date: 2022-05-10 04:15:35 #(use ctrl+Shift+I to insert date string)
# weight: 1
# aliases: ["/first"]
tags: ["SWAG","reverse proxy","Docker","Cloudflare"]
categories: ["tutorials"]
author: "Omar El-Sherif"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
#description: ""
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: false
ShowBreadCrumbs: true
ShowPostNavLinks: true

cover:
    image: "img/cover.png" # image path/url
    alt: "There should be an image here..." # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
editPost:
    URL: "https://github.com/wlanut/walnuthomelab-web/tree/master/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

Recently I decided I wanted to start hosting a blog. If you're reading this post, I was successful in that endeavor. While working out the details, I decided for sure that I wanted it to be a self-hosted operation. I quickly fell down the rabbit hole of researching various blogging platforms and static site generators.

TL;DR This is essentially what I want:

- walnuthomelab.com -> server A (LAN)
- service.walnuthomelab.com -> server A (LAN)
- blog.walnuthomelab.com -> server B (VPS)
- oelsherif.com -> server B (VPS)

## Choices (Yup)

Dedicated blogging platforms:

- **WordPress** \- A very capable platform, but much more than what I needed, and I didn't want to deal with keeping it secure and up to date
- **Ghost** \- A less bloated WordPress alternative, but still felt like too much, and again I didn't want to worry so much about security or updates
- **SSG** \- Static Site Generators create a full static HTML website based on raw data. There is no database to interact with, and since it's a static page there's an added security benefit, as there isn't a plethora of plugins to install and maintain. There were a multitude of options available: Hugo, Gatsby, Jekyll, Next.js, etc. However, Hugo stood out to me the most.

## The Chosen One

So now I had my blogging platform chosen. [Hugo](https://gohugo.io/) works by creating a project directory using Hugo commands, and then creating your posts/content in the form of markdown files. It can then convert that project into an HTML website (based on a theme) and serve it using a built-in live-updating web server. This is awesome, as it means I don't have to spin up a separate Nginx container just to serve the generated site, I can simply point it to my reverse proxy and it'll be available at my chosen subdomain, blog.walnuthomelab.com.

After learning [how to operate Hugo from a container and pointing it at my reverse proxy](https://redhotice.net/posts/hugo-and-swag/), I decided I wanted to migrate the hosting infrastructure to the cloud; it works 100% fine locally, but I'm making an effort to start moving public facing things outside my LAN when practical/possible. To do that, I spun up a cloud instance on Linode.

While doing this, I realize I should probably have a professional-ish landing page for myself; it can house my resume and experience, an 'About Me' page, and link to my blog/Github/LinkedIn, etc. I decided to go with first-initial-last-name, so oelsherif.com.

## Choices Pt. 2

Now I'm at a bit of a dilemma. I want to continue using walnuthomelab.com at home (**Server A**), because it's for my homelab services. But I still want the blog to be located at blog.walnuthhomelab.com, because it's about my homelab, but also on the Linode instance (**Server B**), since it's public facing. I also want oelsherif.com on the Linode instance, because it is also public facing.

For example, I host [Radarr](https://radarr.video/) at home. I *could* just access it from the IP address and port, but that gets kind of annoying after a while. I could just create a DNS record for it so I can use a domain name, but then I have to see that horrid "Not secure" message at the top of my browser. I'd like to have that nice SSL "locked" icon and a valid domain to go to. The solution to this is a reverse proxy.

I set everything up using [SWAG Reverse Proxy with wildcard certs and DNS validation](https://docs.linuxserver.io/general/swag). At its core, SWAG is simply an Nginx reverse proxy that is paired with LetsEncrypt and Fail2Ban, packaged into a neat little Docker container. This way, there's no need to create individual CNAME records at my DNS provider, and my services will be available at service.walnuthhomelab.com.

So once again, this is what we're trying to accomplish:

- walnuthomelab.com -> server A (LAN)
- service.walnuthomelab.com -> server A (LAN)
- blog.walnuthomelab.com -> server B (VPS)
- oelsherif.com -> server B (VPS)

## Step 1: Swag (Server A)

Docker compose file:

```yaml
version: "3.8"
networks:
  walnut01:
    driver: bridge
services:
  swag:
    image: ghcr.io/linuxserver/swag
    container_name: swag
    networks:
      - walnut01
    cap_add:
      - NET_ADMIN
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=$TZ
      - URL=yourdomain.com
      - SUBDOMAINS=wildcard
      - VALIDATION=dns
      - DNSPLUGIN=cloudflare
#      - STAGING=true	
    volumes:
      - ${DOCKERCONFDIR}/swag:/config
    ports:
      - 443:443
      - 80:80
    restart: unless-stopped
```

Make sure to make the appropriate changes in the respective .ini file under `/name-of-swag-volume/dns-conf/`, which in my case is cloudflare.ini. You'll need to input your cloudflare email and Global API key.

Run `docker-compose up -d` (the -d flag means detached, so it won't show all the additional output you may not want to see) and verify that Swag starts successfully and pulls your certs, check yourdomain.com and see if it puts you at the Swag default landing page. You can check the logs by running `docker logs swag`.

## Step 2: Cloudflare (DNS)

In Cloudflare DNS:

1.  Create an A record pointing 'yourdomain.com' to the public IP of Server A
2.  Create CNAME for 'www' pointing to 'yourdomain.com'
3.  Create CNAME for '*' pointing to 'yourdomain.com'
4.  Create an A record pointing 'subdomain.yourdomain.com' to the public IP of Server B

This way, any services you don't have a CNAME record for will be handled by the wildcard cert, pointing to your domain at Server A. You then create that last additional A record for any additional subdomains you specifically want pointed at Server B, as those will take precedence.

## Step 3: Swag (Server B)

Essentially, repeat Step 1 on the cloud server, using the new domain. Once again, verify success by visiting yourdomain2.com and seeing if you hit the SWAG landing page. To add support for the second domain, the first thing we'll need to do is provide SWAG with the ability to utilize valid certificate files.

### Certificates

Try as I might, I could not get Swag to automatically pull a second set of certs for another domain. As a proof of concept to make sure it would actually work, I simply downloaded the working certificate files from Swag's LetsEncrypt folder on Server A, and uploaded them to a folder in Server B. You can find certificate files generated by Swag (or rather Certbot) under `/path/to/swag-config/keys/letsencrypt`. For example:

```nginx
swag
|--keys
  |--letsencrypt
    |--fullchain.pem
    |--privkey.pem
```

In theory, you should only need `fullchain.pem` and `privkey.pem`, but you can copy the whole folder if you wish. Once you have acquired these, you can upload them to Server B. I placed mine in a folder directly under the Swag config root folder, so on the same level as where the 'keys' folder is in the diagram above. Server B example.

```nginx
swag
|--keys
  |--letsencrypt
    |--fullchain.pem
    |--privkey.pem
|--letsencrypt_whl
  |--fullchain.pem
  |--privkey.pem
```

I did it this way because I wanted to make sure the cert files stayed with the rest of the Swag instance, but placed it at the root level because I didn't want Swag to blow out those files as part of any of its automated processes.

**NOTE:** I'm sure there are multiple ways of doing this, and I'm not well versed enough to know what the best way is. Probably rolling a separate Certbot instance. In the future I'll either do this, or figure out a way to just have Ansible rsync the certs over on a weekly cron job.

### Nginx Config

Next, we need to ensure Swag has a way to reference the second set of certificates. If you look under `/swag/nginx/`, you'll see a bunch of '.conf' files. The Nginx portion of Swag loads and uses these to make things "work". We'll need to create a copy of the file named `ssl.conf`. I named mine `ssl_whl.conf`.

Once copied, open the new file, and you'll see a section labeled #Certificates, like so:

```nginx
# Certificates
ssl_certificate /config/keys/letsencrypt/fullchain.pem;
ssl_certificate_key /config/keys/letsencrypt/privkey.pem;
# verify chain of trust of OCSP response using Root CA and Intermediate certs
ssl_trusted_certificate /config/keys/letsencrypt/fullchain.pem;
```

We simply need to change the file paths to match our newly uploaded certificates. In my case, the #Certificates section of my `ssl_whl.conf` file looks like this:

```nginx
# Certificates
ssl_certificate /config/letsencrypt_whl/fullchain.pem;
ssl_certificate_key /config/letsencrypt_whl/privkey.pem;
# verify chain of trust of OCSP response using Root CA and Intermediate certs
ssl_trusted_certificate /config/letsencrypt_whl/fullchain.pem;
```

Next, we'll need to make sure the `ssl_whl.conf` file is loaded for the services we want to use that domain for.

If you look under `/config/nginx/proxy-confs/`, you'll see a bunch of files that have 'subdomain.conf.sample' at the end. These are all preconfigured files for commonly used self-hosted services, like Radarr, Sonarr, and Sabnzbd. Whenever you want Swag to point at one of your services, you just remove the '.sample' part and you're off to the races. If we take a look at any of these, near the top-middle section of the config, we'll see these lines:

```nginx
server_name service.*

include /config/nginx/ssl.conf;
```

If you want a particular service to be pointed at your second domain and use your second set of certificates, change the lines as needed. You'll want the server_name line to be more specific, Swag uses the wildcard with some fancy magic to automatically point at the main domain you first set things up with, so I just type the full intended domain name out. For the next line, simply change ssl.conf to the name of your new ssl file. For hosting my blog at walnuthomelab.com, my subdomain.conf file looks like this:

```nginx
server_name blog.walnuthomelab.com;

include /config/nginx/ssl_whl.conf;
```

**NOTE:** If you want more specific instructions specifically on setting up Hugo with Swag, follow the [guide](https://redhotice.net/posts/hugo-and-swag/) I linked earlier, it's really good.

For my landing page, I just used the base domain, the config looks exactly the same as the one above, but with `server_name oelsherif.com`. Technically you could also tell Hugo to generate the HTML files in your Swag's /www folder, but this felt like a cleaner way of doing it and it works perfectly fine.

## Wrap Up

Assuming you followed all the steps properly, and assuming I didn't forget to mention anything (admittedly I wrote a lot of this post about a month after implementing these changes), you should now be able to use Swag to connect your services to multiple domains on the same server.

The key points:

1.  Make the correct DNS changes (A record pointing desired subdomain to IP of Server B)
2.  Make the valid certificate files for your second domain available to Swag on Server B
3.  Create an additional ssl.conf file to point at those certificate files
4.  Create/adjust your `service.subdomain.conf` files to use the correct ssl.conf and server_name parameters.

If any information in this post ends up being wrong or misleading, don't hesitate to reach out to me to fix it.