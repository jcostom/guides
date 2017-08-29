# Containerized LibreNMS install with SSL reverse proxy

This guide grew out of a combo of a [Reddit post](https://www.reddit.com/r/networking/comments/6usj21/prebuilt_librenms_oxidized_config_backup_rsyslog/) and my homelab setup, where I've installed LibreNMS using Docker on an Ubuntu Linux VM, running on an ESXi 6.5 host. In our setup, the only externally-exposed networking is for the nginx reverse proxy we'll employ to provide a single means of accessing everything we're installing.

Since we're not exposing a bunch of ports outside the host, how do we accomplish the magic of cross-container interactions without lots of extra work (like container IPAM, etc.) then? We'll use a custom Docker bridge, which makes use of Docker's embedded DNS services to support inter-container communication.

**NOTE**: Anything in this document that's enclosed in angle brackets, <like this> should be replaced with something relevant to you/your installation.

## Getting started - Install Ubuntu VM

I installed an Ubuntu 16.04-LTS Server VM. I chose the 64-bit image, since as course of normal operations, I install 64-bit systems whenever possible, to maximize scalability.  I won't walk through the whole process, as it's straight forward enough. I installed with the openssh-server role, and a static IP address.

Do your updates, get your environment setup the way you like it. Shell, aliases, whatever floats your boat. Then come back and get going again...

## Add Docker

Pay a visit to the Docker CE (Community Edition) [website](https://get.docker.com), and do what the script says. I always add my user to the docker group, as suggested at the end of the Docker CE installation process. At the time I'm writing this, the procedure looks like:

```
$ curl -fsSL get.docker.com -o get-docker.sh
$ sh get-docker.sh
$ sudo usermod -aG docker <myusername>
```

After this, I'll logout and back in, so my group membership will be up to date, and I'll be able to execute docker commands.

## Create new Docker bridge network

My prior drafts of this guide made extensive use of Linked Containers, which automagically slip in hosts files entries in containers. It was pointed out to me that linked containers were considered deprecated, and so I've moved away from those, instead favoring a custom Docker bridged network. Why? Non-default bridges in Docker offer embedded DNS services to the containers, so you can have inter-container communication based on the containers' names, which magically turn into DNS entries managed by Docker without you having to do anything about it.

Here, we'll create a simple bridge that mimics the default bridge. Docker will automatically choose the next /16 from 172.16/12, which will be 172.18/16 as our subnet.

```
$ docker network create \
    -o "com.docker.network.bridge.name"="docker1" \
    -o "com.docker.network.bridge.enable_ip_masquerade"="true" \
    -o "com.docker.network.bridge.enable_icc"="true" \
    -o "com.docker.network.bridge.host_binding_ipv4"="0.0.0.0" \
    -o "com.docker.network.driver.mtu"="1500" \
    containers
```

You can verify that it's setup the way you expect with `docker inspect containers`.

What's the dark side of this? Now we'll need to specify `--net=containers` on all of our containers that we'd like to be part of this new bridged network.

## Install Portainer

[Portainer](https://portainer.io) is a nice user interface for quick access to basic operations and telemetry from a Docker host.  We're installing it to manage the location instance of Docker, though it's capable of managing a swarm cluster as well. That's outside the scope of what we're doing, but it's good to know it's possible, right?

```
$ docker run -d \
    --restart=unless-stopped \
    --name portainer \
    --net=containers \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /var/docks/portainer:/data \
    portainer/portainer
```

## Install MariaDB

MariaDB is an open source alternative to MySQL, which is built to be fully compatible with MySQL. So, we'll go with that. You'll note that the container takes the root password as a parameter when instantiating the container. I generated the password using 1Password's generation utility.

PUID and PGID are the uid and gid values you want the processes to run as. For me, this was my normal user, though you're free to use whatever makes sense for your deployment.

```
$ docker run -d \
    --restart=unless-stopped \
    --name=mariadb \
    --net=containers \
    -e PUID=1000 \
    -e PGID=1000 \
    -e MYSQL_ROOT_PASSWORD=<root-db-password> \
    -e TZ=America/New_York \
    -v /var/docks/mariadb:/config \
    linuxserver/mariadb
```

Before we move on to setup anything else, we'll create the database we're going to end up using in our LibreNMS installation, as well as create the needed user.  We'll launch a shell inside the container, create the database, the user, and finally set the password to allow access from the private Docker network that's not exposed outside the host.

```
$ docker exec -it mariadb /bin/bash
# mysql -p<root-db-password>
```

Now that you're connected, create the database & user. Again, generate a strong password somehow.

```
CREATE DATABASE librenms CHARACTER SET utf8 COLLATE utf8_unicode_ci;
CREATE USER 'librenms'@'172.18.0.0/255.255.0.0' IDENTIFIED BY '<librenms-db-passwd>';
GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'172.18.0.0/255.255.0.0';
FLUSH PRIVILEGES;
exit
```

Now that you've done that, exit that shell and get back to the host.

## Setup Oxidized

```
$ docker run -d -t \
    --name=oxidized \
    --restart=unless-stopped \
    --net=containers \
    -v /var/docks/oxidized:/etc/oxidized \
    alectolytic/oxidized
```

At this point, you'll want to stop the Oxidized container so you can configure it. You do this, of course, with `$ docker stop oxidized`. Here's a sample `/var/docks/oxidized/config`:

```
---
model: junos
interval: 3600
use_syslog: false
debug: false
threads: 30
timeout: 20
retries: 3
prompt: !ruby/regexp /^([\w.@-]+[#>]\s?)$/
rest: 0.0.0.0:8888
next_adds_job: false
vars: {}
groups:
  jnpr:
    username: <username used on Juniper devices>
    password: <password used on Juniper devices>
    model: junos
pid: "/var/run/oxidized/pid"
input:
  default: ssh
  debug: false
  ssh:
    secure: false
output:
  default: git
  file:
    directory: "/etc/oxidized/configs"
  git:
    single_repo: true
    user: <Oxidized User>
    email: <someemail@somewhere.com>
    repo: "/etc/oxidized/configs.git"
source:
  default: csv
  csv:
    file: "/etc/oxidized/router.db"
    delimiter: !ruby/regexp /:/
    map:
      name: 0
      model: 1
      group: 2
model_map:
  cisco: ios
  juniper: junos
```

Further, here's a sample `/var/docks/oxidized/router.db` file:

```
router1:juniper:jnpr
router2:juniper:jnpr
switch1:juniper:jnpr
```

At this point, you can restart Oxidized with `$ docker start oxidized`.

## Install LibreNMS

Before you get cooking, you'll want to create the config.custom.php file in advance. This is where you keep any config statements that are "local" in nature, ie not contained in the standard config file that would get wiped if you rebuilt the container. You'll want to create the directory first: `$ sudo mkdir /var/docks/librenms`.  Next, create the file `/var/docks/librenms/config.custom.php`. Here's a sample:

```
// config.custom.php

$config['bad_if_regexp'][] = '/^docker[\w]+$/';
$config['bad_if_regexp'][] = '/^lxcbr[0-9]+$/';
$config['bad_if_regexp'][] = '/^veth.*$/';
$config['bad_if_regexp'][] = '/^virbr.*$/';
$config['bad_if_regexp'][] = '/^lo$/';
$config['bad_if_regexp'][] = '/^macvtap.*$/';
$config['bad_if_regexp'][] = '/gre.*$/';
$config['bad_if_regexp'][] = '/tun[0-9]+$/';

$config['snmp']['community'] = array("<default-snmp-community>");
$config['enable_printers'] = 1;
$config['mydomain'] = 'domain.org';
#$config['front_page'] = "pages/front/map.php";
$config['geoloc']['latlong'] = TRUE;
$config['geoloc']['engine'] = "google";
$config['map']['engine'] = "leaflet";
$config['leaflet']['default_lat'] = '<latitude>';
$config['leaflet']['default_lng'] = '<longitude>';
$config['leaflet']['default_zoom'] = 12;

$config['update'] = 1;

$config['oxidized']['ignore_types'] = array('server','printer','wireless','storage');
$config['oxidized']['ignore_os'] = array('linux');
```

Got it? Ok, create the container.  The hostname you use in the BASE_URL should match the address you use to access the host. You'll want this to match what's in the nginx configuration we'll do later.

```
$ docker run -d \
    --restart=unless-stopped \
    --name=librenms \
    --net=containers \
    -e DB_HOST=mariadb \
    -e DB_NAME=librenms \
    -e DB_USER=librenms \
    -e DB_PASS=<librenms-db-passwd> \
    -e BASE_URL=https://<hostname-youre-using.domain.org> \
    -e POLLERS=16 \
    -e TZ=America/New_York \
    -v /var/docks/librenms/logs:/opt/librenms/logs \
    -v /var/docks/librenms/rrd:/opt/librenms/rrd \
    -v /var/docks/librenms/config.custom.php:/opt/librenms/config.custom.php:ro \
    jarischaefer/docker-librenms
```

Next, we'll run a couple of commands to get the database populated, a user created, and then get the librenms version back on master, update the app and db schema.

```
$ docker exec librenms sh -c "cd /opt/librenms && php /opt/librenms/build-base.php"
$ docker exec librenms php /opt/librenms/adduser.php admin admin 10 test@example.com
$ docker exec -it librenms /bin/bash
# cd /opt/librenms
# git checkout master
# git pull
# php includes/sql-schema/update.php
# exit
```

At this point, you can't login quite yet - that's going to happen after the next step, where we setup the nginx reverse proxy.

## Install nginx

First, create the container, then stop the container so we can complete our configuration. PUID and PGID are the uid and gid values you want the processes to run as. For me, this was my normal user, though you're free to use whatever makes sense for your deployment.

```
$ docker run -d \
    --name=nginx \
    --restart=unless-stopped \
    --net=containers \
    -v /var/docks/nginx:/config \
    -e PGID=1000 -e PUID=1000 \
    -p 80:80 -p 443:443 \
    -e TZ=America/New_York \
    linuxserver/nginx
$ docker stop nginx
```

Next, pop on over to `/var/docks/nginx` and make some changes. We'll start with creating some Diffie-Hellman Key Exchange parameters. [Here's a better explanation](https://security.stackexchange.com/questions/94390/whats-the-purpose-of-dh-parameters) than I'd write about what it's for.

```
$ openssl dhparam -out /var/docks/nginx/keys/dhparam.pem 2048
```

Next, you'll want to create a directory for some configuration snippets we'll use: `$ mkdir /var/docks/nginx/snippets`.  We'll use 4 files for our configuration snippets (put them in the snippets directory). In the internal-ip.conf file, you can list internal subnet(s) you'll be coming from. I included this mainly since you wouldn't necessarily want external parties gaining access to Portainer or LibreNMS. Naturally, you can take this much further and integrate additional access controls, but that's out of scope for this guide.

```
$ cat internal-ip.conf
allow <your internal subnet>/24;
allow 172.18.0.0/16;
deny all;

$ cat librenms.conf
location / {
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_pass http://librenms/;
    include /config/snippets/internal-ip.conf;
}

$ cat portainer.conf
location /docks/ {
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_pass http://portainer:9000/;
    include /config/snippets/internal-ip.conf;
}
location /docks/api/websocket/ {
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_http_version 1.1;
    proxy_pass http://portainer:9000/api/websocket/;
    include /config/snippets/internal-ip.conf;
}

$ cat ssl-params.conf
ssl_protocols TLSv1.1 TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
ssl_ecdh_curve secp384r1;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
ssl_dhparam /config/keys/dhparam.pem;
```

Lastly, the remainder of the configuration:

```
$ cat /var/docks/nginx/nginx/site-confs/default
server {
    listen 80 default_server;
    server_name <hostname-youre-using.domain.org>;
    return 302 https://$server_name$request_uri;
}

server {
    listen 443 ssl default_server;
    server_name <hostname-youre-using.domain.org>;

    ssl_certificate /config/keys/cert.crt;
    ssl_certificate_key /config/keys/cert.key;

    include /config/snippets/ssl-params.conf;
    include /config/snippets/portainer.conf;
    include /config/snippets/librenms.conf;
}
```

Got it? Ok, start up nginx, and you're done. `$ docker start nginx`

LibreNMS will be found at `https://<hostname-youre-using.domain.org/`, and Portainer will be found at `https://<hostname-youre-using.domain.org/docks/`.

## Login, Play

Hit `https://<hostname-youre-using.domain.org/`, login as admin/admin, and get to work. Obviously, you'll want to make a different admin user, and obliterate the admin/admin one. Unless of course, you were the forward-thinking, free-spirit type that didn't use admin/admin way back when you added a user to LibreNMS. ;-)

When you pop into the options to add Oxidized support, the URL you want is `http://oxidized:8888/`.
