---
title: "Rundeck Installation Tips"
date: 2022-06-24 23:56:14 #(use ctrl+Shift+I to insert date string)
# weight: 1
# aliases: ["/first"]
tags: ["Rundeck","Automation","Ansible"]
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

# Installation Options

In this post, I'm going to cover **2** methods for installing and configuring Rundeck: **Docker** installation, and **Local** installation.

During the process of my initial **Docker** installation, I was able to get Rundeck in a "working" state, (functioning projects/jobs/commands/nodes, etc.), at least until I wanted to use the Ansible plugin. Rundeck with Ansible integration requires Ansible to be installed on the same "server", so if we have Rundeck in a container, we would need Ansible to be installed within that container. I couldn't figure out how to do this, and it seemed rather pointless as it's not very well documented, so I ended up with the **Local** server installation method, using a Postgres container as my DB.

## Method 1 - Docker Installation

### Docker-Compose Setup:

- [Docker-compose example for basic setup](https://github.com/rundeck/docker-zoo/blob/master/basic/docker-compose.yml)
- [Docker-compose example for postgres db](https://github.com/rundeck/docker-zoo/blob/master/postgres/docker-compose.yml)
- Adjust DB username/password as desired, leave `POSTGRES_DB=rundeck` alone though
- Can add persistent local storage bind volume for postgres db like so:

```yaml
- ${DOCKERCONFDIR}/rundeck/postgres/data:/var/lib/postgresql/data
```

My Docker-Compose Example:

```yaml
rundeck:
    image: rundeck/rundeck:4.3.0
    container_name: rundeck
    links:
      - postgres
    environment:
        RUNDECK_DATABASE_DRIVER: org.postgresql.Driver
        RUNDECK_DATABASE_URL: jdbc:postgresql://postgres/rundeck
        RUNDECK_DATABASE_USERNAME: omelsherif
        RUNDECK_DATABASE_PASSWORD: ${DB_PASS}
        RUNDECK_GRAILS_URL: http://192.168.1.20:4440
    volumes:
      - ${RUNDECK_LICENSE_FILE:-/dev/null}:/home/rundeck/etc/rundeckpro-license.key
      - ${DOCKERCONFDIR}/rundeck/server/resources.yaml:/home/rundeck/server/data/resources.yaml
    ports:
      - 4440:4440
    restart: always
  
  postgres:
    image: postgres
    container_name: postgres
    expose:
      - 5432
    environment:
      - POSTGRES_DB=rundeck
      - POSTGRES_USER=omelsherif
      - POSTGRES_PASSWORD=${DB_PASS}
    volumes:
      - ${DOCKERCONFDIR}/rundeck/postgres/data:/var/lib/postgresql/data
    restart: always
```

Things I changed from the official examples (linked above):

- You may want to add "http://" in front of your RUNDECK\_GRAILS\_URL if you're just accessing it from localhost or from an IP without SSL, otherwise you may run into the error described in [this GitHub issue](https://github.com/rundeck/rundeck/issues/4417) like I did

- If accessing Rundeck from outside its VM/server, you'll want to change the RUNDECK\_GRAILS\_URL value from "localhost" to the IP of your server

- It's a good idea to use a .env variable for your DB password so it's not visible in your compose file

- The examples show Docker volumes created by Docker, rather than bind volumes on disk; I switched to disk since that's what I normally use

Run `docker-compose up -d` and Rundeck should be available at the URL you defined.

## Method 2 - Local Installation

You can follow the [Rundeck documentation](https://docs.rundeck.com/docs/administration/install/linux-deb.html#installing-rundeck) for some simple Linux installation instructions on installing Rundeck itself.

### Choosing a database

Rundeck [supports several database options](https://docs.rundeck.com/docs/administration/configuration/database/#customize-the-datasource). I decided to use Postgres; I don't have a particular reason other than "it's what the cool kids use." That is to say people complain a lot about MySQL, MariaDB seems to be more favorable than that, and Postgres the most favorable. Maybe one day I'll understand why. In any case, you can then choose to set up your database using either a locally installed instance, or a container. I decided to go with a container as I like to keep things segregated when possible.

### Postgres - Docker Container

I used the official Postgres container during my install. Here is my Docker-Compose snippet:

```yaml
postgres:
    image: postgres
    container_name: postgres
    environment:
      - POSTGRES_DB=rundeck
      - POSTGRES_USER=rundeck
      - POSTGRES_PASSWORD=${DB_PASS}
    volumes:
      - ${DOCKERCONFDIR}/postgres/rundeck:/var/lib/postgresql/data
    ports:
      - 5432:5432
    restart: always
```

Once the container is up and running, you can connect to it from Rundeck by modifying the configuration file located at /etc/rundeck/rundeck-config.properties. You'll want to reference the [Rundeck/Postgres documentation](https://docs.rundeck.com/docs/administration/configuration/database/postgres.html#install-postgresql), but mine currently looks like this:

```ini
grails.serverURL=http://192.168.1.20:4440
dataSource.driverClassName = org.postgresql.Driver
dataSource.url = jdbc:postgresql://192.168.1.20/rundeck
dataSource.username = rundeck
dataSource.password = DB_PASS
dataSource.dbCreate = update
```

In the 3rd line, replace the IP address with either the hostname or IP of the server hosting your Postgres DB. Since I'm using a container, and have port 5432 in the container mapped to port 5432 on my host, I can either use "localhost", the loopback address, or the IP of my server to make the connection. Once you've made the necessary changes, run `sudo service rundeckd restart` again to restart Rundeck with the new configuration.

NOTE: Check out [this guide](https://www.postgresqltutorial.com/postgresql-getting-started/connect-to-postgresql-database/) for some good examples of how to connect to your Postgres server once it's up to confirm it's reachable and working.

### Postgres - Local DB Install

There are some good instructions for installing a local Postgres DB instance [here](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-20-04). Once again, you can combine that with the [Postgres Rundeck documentation](https://docs.rundeck.com/docs/administration/configuration/database/postgres.html#install-postgresql) to connect Rundeck to the database. I often piece together what I need from multiple sources these days. I haven't followed the local DB installation process to completion, but it should be relatively straightforward.

## Logging In

Once Rundeck is up and running, you can reach the login interface. If you haven't change the default credentials, they will be admin/admin.

### Changing default credentials - Docker

From the documentation:

"The default setup utilizes the /home/rundeck/server/config/realm.properties file. Mount or otherwise replace this file to manage further users through this method."

So basically you'll want to create a file named `realm.properties` and mount it with a bind volume, so it remains persistent, similar to how we did the resources.yaml file above.

### Changing default credentials - Local

- `nano /etc/rundeck/realm.properties` (or use your editor of choice)
- Find the line `admin:admin,user,admin,architect,deploy,build`
- Change to `admin:YourNewPassword,user,admin,architect,deploy,build`

The default realm.properties file (with changed password) looks something like this:

```ini
#
# This file defines users passwords and roles for a HashUserRealm
#
# The format is
#  <username>: <password>[,<rolename> ...]
#
# Passwords may be clear text, obfuscated or checksummed.  The class
# org.mortbay.util.Password should be used to generate obfuscated
# passwords or password checksums
#
# If DIGEST Authentication is used, the password must be in a recoverable
# format, either plain text or OBF:.
#
#jetty: MD5:164c88b302622e17050af52c89945d44,user
#admin: CRYPT:ad1ks..kc.1Ug,server-administrator,content-administrator,admin
#other: OBF:1xmk1w261u9r1w1c1xmq
#plain: plain
#user: password
# This entry is for digest auth.  The credential is a MD5 hash of username:realmname:password
#digest: MD5:6e120743ad67abfbc385bc2bb754e297

#
# This sets the default user accounts for the Rundeck app
#
admin:YourNewPassword,user,admin,architect,deploy,build

#
# example users matching the example aclpolicy template roles
#
#job-runner:admin,user,job_runner
#job-writer:admin,user,job_writer
#job-reader:admin,user,job_reader
#job-viewer:admin,user,job_viewer
```

## Projects

[Project creation documentation](https://docs.rundeck.com/docs/manual/projects/configuration.html)

[SSH Node Execution settings](https://docs.rundeck.com/docs/manual/projects/node-execution/ssh.html)

[SSH keygen instructions](https://docs.rundeck.com/docs/manual/projects/node-execution/ssh.html#ssh-key-generation)

> Rundeck only accepts keys generated with RSA, use `ssh-keygen -t rsa -b 4096 -m PEM`

## Nodes

[Setting up Nodes/SSH](https://docs.rundeck.com/docs/learning/howto/ssh-on-linux-nodes.html)

The resources.yaml file just needs to be within a directory that the Rundeck container/user can reach. If using Docker, you can browse the container files in VS Code with the Docker plugin to see possible locations, and then use a Docker bind volume to mount your resources file so it remains persistent, and point to it in the configuration. Example bind mount:

```yaml
${DOCKERCONFDIR}/rundeck/server/resources.yaml:/home/rundeck/server/data/resources.yaml
```

You can also use the URL option and point it to the URL of a file on a GitHub repo.

Sample resources.yaml block:

```yml
remote-node: #treat this as the displayname
  description: Remote SSH server node
  hostname: node1 #IP address or DNS name
  nodename: node1
  osArch: amd64
  osFamily: unix
  osName: Linux
  osVersion: 5.11.0-7612-generic
  tags: 'node1'
  username: agent #ssh username (omelsherif in my case)
  ssh-key-storage-path: keys/id_rsa #path to ssh key in key storage
  ssh-key-passphrase-storage-path: keys/id_rsa-pass #use if your key has a passphrase
  sudo-password-storage-path: keys/sudo-pass #use if your jobs need sudo
```

Once you have a remote node configured, you can verify connectivity with a [simple command](https://docs.rundeck.com/docs/learning/howto/ssh-on-linux-nodes.html#running-commands-on-nodes).

## Commands & Jobs

You can review the Rundeck documentation for examples/setup of [Commands](https://docs.rundeck.com/docs/manual/06-commands.html#commands-tab-overview) and [Jobs](https://docs.rundeck.com/docs/manual/04-jobs.html#overview).

## Configuring SSL

So you've got everything up and running, you've created a project, added some nodes, ran some commands, and set up a few jobs. Everything seems to be working, but now you want to make things a touch more secure, maybe use a nice domain/subdomain to access Rundeck from.

So let's configure Rundeck to use SSL! What I'm going to cover here is simply what I did, and what worked for me. I am currently using the [SWAG container](https://docs.linuxserver.io/general/swag) to handle my certificates and reverse-proxy (Nginx), you can refer to the above link for setup instructions if you're interested. 

I used a couple sources to assist in getting things working:

- [Blog Post 1](https://yallalabs.com/automation-tool/how-to-configure-nginx-with-ssl-as-a-reverse-proxy-for-rundeck/)
- [Blog Post 2](https://geekdudes.wordpress.com/2018/12/14/rundeck-nginx-as-ssl-reverse-proxy/)

These blog posts mostly match up in terms of the configuration, but there were some slight differences. I'm now going to explain what worked for me.

### \> Nginx

Nginx conf file:

```nginx
## Version 2021/05/18
# make sure that your dns has a cname set for <container_name> and that your <container_name> container is not using a base url

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name rundeck.walnuthomelab.com; # Replace with your subdomain

    include /config/nginx/ssl.conf;

    client_max_body_size 0;

    location / {
        #add_header          Front-End-Https on;
        proxy_set_header    Host $host;
        proxy_set_header    X-Real-IP $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto $scheme;

        proxy_pass          http://192.168.1.20:4440;
        proxy_read_timeout  90;

        proxy_redirect      http://192.168.1.20:4440 https://rundeck.walnuthomelab.com;
        }
}


server {
    listen 80;
    server_name rundeck.walnuthomelab.com;
    return 301 https://$host$request_uri;
}
```

As you can see from the comment near the top, replace all instances of my domain name with your own. You will also notice the line that says `include /config/nginx/ssl.conf;` which is an additional file that SWAG generates for the SSL stuff. For the sake of readability, I'll post the contents here as well:

```nginx
ssl_session_timeout 1d;
ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions
ssl_session_tickets off;

# intermediate configuration
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;

# OCSP stapling
ssl_stapling on;
ssl_stapling_verify on;


### Linuxserver.io Defaults

# Certificates
ssl_certificate /config/keys/letsencrypt/fullchain.pem;
ssl_certificate_key /config/keys/letsencrypt/privkey.pem;
# verify chain of trust of OCSP response using Root CA and Intermediate certs
ssl_trusted_certificate /config/keys/letsencrypt/fullchain.pem;

# Diffie-Hellman Parameters
ssl_dhparam /config/nginx/dhparams.pem;

# Enable TLS 1.3 early data
ssl_early_data on;
```

If you're not using SWAG, you can simply refer to the SSL config of the blog posts I linked above. If for some reason those posts have gone missing, using my SSL config \*should\* work as long as you point it to the correct certificate files. It is all just Nginx after all.

### \> Rundeck Config

Open `/etc/rundeck/framework.properties` and make the following modifications:

```bash
nano /etc/rundeck/framework.properties
```

```ini
# ----------------------------------------------------------------
# Rundeck server connection information
# ----------------------------------------------------------------

framework.server.name = rundeck.walnuthomelab.com
framework.server.hostname = rundeck.walnuthomelab.com
framework.server.port = 4440
framework.server.url = https://rundeck.walnuthomelab.com
```

Replace the domains with your own where applicable.

Next, open `/etc/rundeck/rundeck-config.properties` and make the following modifications:

```bash
nano /etc/rundeck/rundeck-config.properties
```

```ini
# change hostname here
grails.serverURL=https://rundeck.walnuthomelab.com
```

Next, refer to the [Rundeck documentation](https://docs.rundeck.com/docs/administration/security/ssl.html#using-an-ssl-terminated-proxy) for some further instructions instructions on how to ensure SSL is being listened for.

Finally, restart both Rundeck, and whatever form of Nginx you're using:

```bash
/etc/init.d/rundeckd restart
#If using local Nginx
Systemctl restart nginx
#If using Docker/Swag
docker restart nginx
#or
docker restart swag
```

Make sure to double check your Nginx logs to verify that you have everything configured correctly and that there are no errors, then visit your subdomain to see if everything works!

## Wrap Up

There's a whole lot more to Rundeck than I've explained in this post. I'm still getting the hang of things, especially Ansible integration. But I hope that if anyone is having issues with getting the damn thing running in the first place, they can glean some useful information from this blog post.
