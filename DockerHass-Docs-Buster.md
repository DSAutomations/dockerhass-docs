



## Overview
Years ago as a new homeowner, I was starting my home automation journey. With an with an eye on the budget, I was drawn into the Home Assistant community. I found the do-it-yourself approach favorable to being locked into commercial cloud ecosystem. Before I knew it, I had opted to host HA on a Raspberry Pi 3b, on my Ubiquiti network consisting of an EdgeRouterX and Unifi access points. 

When I first deployed my setup, Hass.io was only in its infancy, however the hassbian image seemed to be a reasonable option. Time went on, and I threw a few hours into development every week and I was always upgrading, adding on, and exploring new integrations. 

Ultimately, I've built some awesome features and automatons that my family and mysel enjoys, but the Raspberry Pi has become a veritable house of cards. Along with homeassistant, I've installed all sorts of goodies for development, and integration. Nginx, pihole, mosquitto, apcupsd, certbot, awscli, samba, git, and the list goes on.

Like many others, I saw the popup in HA about python 3.5 depreciation recently, and through sweat and tears, managed to rebuild a new venv and get Homeassistant running in it without the warning. However, this came at the cost of a dramatic performance hit in the recorder component. I'm a big fan of graphs on my front end, and now they're all taking twice as long to load. Not sure where I went wrong, but one thing that I am sure about is that there is a better way, install hass.io... 

No, just kidding, but seriously, Hass.io is essentially this what we're doing here, but with a lot of the hard work already done for you. 

However, if you take take that shortcut, you're locking yourself in a box where you'll only be able to use the [hass.io addons](https://www.home-assistant.io/addons/). Manually setting up a Docker stack gives us a far greater degree of control and tweakability. 

I aim to give you a comprehensive guide for getting the *pictured* environment up and running from scratch, this environment includes the following:
* **Homeassistant**
* **Nginx** *reverse proxy for serving http securely*
* **Mosquitto MQTT** *for more features than the built in broker*
* **MariaDB** *to replace the built-in DB for better performance*
* **InfluxDB** *to efficiently capture time-series data*
* **Grafana** *for more robust graphing and dashboards*
* **Node Red** *for flow-based automatons*
* **Portainer** *for web based container management*

## Getting Started
We're starting from the ground up. I'm using a Raspberry Pi 3b. Let's grab the latest [Raspbian Buster Lite](https://downloads.raspberrypi.org/raspbian_lite_latest) and flash it to an SD card using an app like [etcher](https://www.balena.io/etcher/). 

*Quick note on SD cards: As mentioned in a recent [homeassistant blog](https://www.home-assistant.io/blog/2019/06/26/release-95/), not all SD cards are created equal. If you try to store and run databases on a cheap SD card, you can burn it out in a hurry. Pick up a class A1 or A2 card to save yourself some headaches in the future.*

I like to go headless on my Pi, so I'll add two files to my boot partition to make this possible:

    ssh
    wpa_supplicant
`shh` is an empty file and flags the OS to enable SSH on first boot. 

`wpa_supplicant` contains the wifi configuration for my network (WPA2/PSK), and looks like this:

    country=US
    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    update_config=1
    network={
    	ssid="MYSSID"
    	psk="passphrase"
    	scan_ssid=1
    	key_mgmt=WPA-PSK
    	}

Change your [country code](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2#Current_codes) accordingly. 

Once these two files have been moved to the boot partition, eject/dismount the drive and move the SD card to the Raspberry Pi. Power up the Raspberry Pi using a power supply rated for the task, and watch for the device to appear on your wifi. 

Connect to the device via SSH with the default username `pi` and the default password `raspberry` and we're ready to get started. 
##
**Optionally:** You can either directly enter your wifi passphrase into the `wpa_supplicant` file, or for increased security you can give it give it the pre-encoded key. This key is generated with the `wpa_passphrase`tool:
 ```
   $ wpa_passphrase MYSSID passphrase
   network={
   	ssid="MYSSID"
   	#psk="passphrase"
         psk=59e0d07fa4c7741797a4e394f38a5c321e3bed51d54ad5fcbd3f84bc7415d73d
   }
   ```
You would then need to take the generated output and replace the `psk` line in the `wpa_supplicant` file.


## Install Docker

At this point, I'm sitting at a nice fresh terminal on my pi. We need to start with a bit of housekeeping:
```
sudo apt update
sudo apt upgrade
```
Use the raspbian config script to set the hostname and password on your device:
```
sudo raspi-config
```
All of the following packages are required to get Docker and docker-compose installed:
```
sudo apt-get install apt-transport-https \
                      ca-certificates \
                      software-properties-common \
                      build-essential \
                      libssl-dev \
                      libffi-dev \
                      python \
                      python-dev \
                      python-pip 
                      python-backports.ssl-match-hostname -y
```
Next let's add a repository for us to pull the latest docker packages from, and also, we'll make it so that packages signed by the docker project are permitted to run.
```
echo "deb https://download.docker.com/linux/raspbian/ stretch stable" | sudo tee -a /etc/apt/sources.list
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```
With all of this done, we should have everything we need to install docker. This might take a little while, let's go:
```
curl -fsSL get.docker.com -o get-docker.sh
sh get-docker.sh
```

*Note: the above commands should normally work, but with Raspbian Buster fresh off the presses as of the time of this writing, the following kludge is necessary until the `get-docker.sh` is updated by the docker team:*

`sudo curl -sL get.docker.com | sed 's/9)/10)/' | sh`

Now that Docker is installed we'll make things a bit easier by putting our user in the docker group:
```
sudo usermod -aG docker $USER
```
After doing this, you must **log out and back in** for the group permissions to be applied.

Almost done, let's install docker-compose:
```
sudo pip install docker-compose
```
## Docker Images
Images are at the core of docker, we're going to use a set of them to create our environment. You can create your own, but that is beyond the scope of this guide. For today, we're going to use public images from [Docker Hub](https://hub.docker.com). 

The first time we launch our stack, the images will be downloaded and cached on our system. Subsequent runs will utilize these downloaded images.

One important note is that not all images available on Docker Hub will be compatible with the Raspberry Pi. An image needs to be created with a compatible CPU architecture. Here are the images we will be using. I've found these to be reasonably popular images (important) which are compatible with my Raspberry Pi 3b:

* [homeassistant/raspberrypi3-homeassistant](https://hub.docker.com/r/homeassistant/raspberrypi3-homeassistant)
* [jsurf/rpi-mariadb](https://hub.docker.com/r/jsurf/rpi-mariadb)
* [nodered/node-red-docker](https://hub.docker.com/r/nodered/node-red-docker)
* [influxdb](https://hub.docker.com//influxdb)
* [fg2it/grafana-armhf](https://hub.docker.com/r/fg2it/grafana-armhf)
* [portainer/portainer](https://hub.docker.com/r/portainer/portainer)
* [nginx](https://hub.docker.com//nginx)

### Tags
Some images may be directly compatible with the Raspberry Pi, but for others we may need to specify a version. To get a desired version of a container, append a tag to it. The standard syntax is *`i<Imageme:tagna.*  Here is the list of both images *and* tags that we will deploy in the main stack: 
* homeassistant/raspberrypi3-homeassistant
* jsurf/rpi-mariadb:latest
* nodered/node-red-docker:rpi-v8
* influxdb:latest
* fg2it/grafana-armhf:v5.0.4
* portainer/portainer
* nginx


# Volumes
Docker containers are ephemeral. In short, we can start up a container and do work with it, but when it's shut down any data contained within will be lost. We can gain persistence between sessions by mounting volumes which will link directories outside the docker containers to directories within.

Let's get this setup. We can keep our volumes anywhere, but the convention is to keep them in `/srv/`. Let's create a directory and set permissions:

```
sudo mkdir /srv/docker
sudo chown pi:docker /srv/docker
```
Now, let's create the rest of the needed directories:
```
mkdir /srv/docker/homeassistant
mkdir /srv/docker/letsencrypt
mkdir /srv/docker/letsencrypt/live
mkdir /srv/docker/mariadb
mkdir /srv/docker/nodered
mkdir /srv/docker/influxdb
mkdir /srv/docker/grafana
mkdir /srv/docker/portainer
mkdir /srv/docker/nginx
mkdir /srv/docker/nginx/config
mkdir /srv/docker/nginx/ssl
```


## Optional: Setup Samba
You can do all of your config file creation and editing at the command line if you want, however this can be a bit cumbersome. It would be helpful to directly e your config files o your local PC, let's get Samba up and running to provide  service to us. We'll set this up using `docker-compose` the same tool we'll use later for our main stack. This will be a good opportunity for a bit of practice.

You'll notice in the config below, we're declaring an image we want to use with a tag that's specific to the raspberry pi. We're also passing in a list of volumes we want to link in a format like this: 

*`/path/outside/container:/path/inside/container`*

Additionally, there's a list of ports needed by the SMB protocol that we're going to pass though in the same *`outside:inside`* format, as well as a bunch of other attributes needed by docker to setup the container.

To get Samba up and running, c

reate a new folder in your home directory and create a file inside called `docker-compose.yml` 

```
mkdir ~/samba-server
nano ~/samba-server/docker-compose.yml
```
Paste the following config:
```
version: '3.4'
services:
  samba:
    image: dperson/samba:armhf
    ports:
      - "137:137/udp"
      - "138:138/udp"
      - "139:139/tcp"
      - "445:445/tcp"
    volumes:
      - /srv/docker:/srv/docker
      - /etc/localtime:/etc/localtime:ro
    tmpfs:
      - /tmp
    restart: unless-stopped
    command: > 
      -u "smbuser;badpass" 
      -s "docker-config;/srv/docker;yes;no;no;smbuser"
    environment:      
      - 'USERID=1000'
      - 'GROUPID=995'
```
Change `badpass` to something better
Save the file by pressing  `Ctrl-o` then exit with `Ctrl-x`.

*Note: the environmental variables above should work on Raspbian Buster but you may need to adjust USERID or GROUPID if you're on a different system or have added additional users. You can find these values by evoking `id` on the command line. Assign the UID of your user and the GID of docker.*


To start the sharing service, make sure that you're in the `samba-server` directory and bring up the container:
```
cd ~/samba-server
docker-compose up
```
Watch the logs that appear to see if everything is going smoothly. If things looks positive, you should now you should be able to connect to your instance using the username `smbuser` and whatever you changed `badpass` to moments ago. We named the share `docker-config`, so connect to that using the standard SMB convention: 

`\\hostname\docker-config` or `smb://hostname/docker-config/`

You should see the list of directories that we created in the last section. Now would be a good time to copy your Home Assistant config into the `homeassistant` directory.

When you're finished, switch back to the terminal and press `Ctrl-c` to kill the server. Keep this shutdown when you're not using it.  

Once things are working consistently, use `docker-compose up -d` to start the container in the background. After launching a container like this, use `docker-compose down` to stop it again.


## The big stack: Home Assistant

We've got the groundwork down to start building our main stack. We're going to configure our containers using a config file called`docker-compose.yml`.  We'll then use that file to launch all of our containers at once. 

But we need to start small and build up, we'll start with the application we're building our stack around: Home Assistant. The HA developers make things easy for us and published a Raspberry Pi optimized version of the application on Docker Hub.

First thing we need is to create our config file. Let's also put it in its own directory:
 ```
mkdir ~/docker-ha && cd ~/docker-ha
nano docker-compose.yml
```


```
version: '3'
services:
  homeassistant:
    container_name: homeassistant
    restart: unless-stopped
    image: homeassistant/raspberrypi3-homeassistant
    volumes:
      - /srv/docker/HomeAssistantConfig:/config
      - /etc/localtime:/etc/localtime:ro
    ports:
      - 8123:8123
    privileged: true

```





```
version: '3'
services:
  homeassistant:
    container_name: homeassistant
    restart: unless-stopped
    image: homeassistant/raspberrypi3-homeassistant
    volumes:
      - /srv/docker/HomeAssistantConfig:/config
      - /etc/localtime:/etc/localtime:ro
      - /srv/docker/letsencrypt/live:/letsencrypt
    ports:
      - 8123:8123
    privileged: true
    depends_on:
     - mariadb
     - influxdb
  mariadb:
    image: jsurf/rpi-mariadb:latest
    ports:
      - 3306:3306/tcp
    volumes:
      - /srv/docker/mariadb/config:/etc/mysql/conf.d
      - /srv/docker/mariadb/data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=super_database_password
      - MYSQL_DATABASE=homeassistantdb
      - MYSQL_USER=homeassistant
      - MYSQL_PASSWORD=a_database_password
    restart: always
  nodered:
    image: nodered/node-red-docker:rpi-v8
    ports:
     - 1880:1880
    volumes:
     - /srv/docker/nodered:/data
    user: "1000"
    restart: always
    environment:
      TZ: "America/New_York"
    depends_on:
     - homeassistant
  influxdb:
    image: influxdb:latest
    restart: always
    ports:
     - 8086:8086
     - 8083:8083
     - 2003:2003
    volumes:
     - /srv/docker/influxdb/data:/var/lib/influxdb
  grafana:
    image: fg2it/grafana-armhf:v5.0.4
    ports:
     - 3000:3000
    volumes:
     - /srv/docker/grafana/data:/var/lib/grafana
    restart: always
    depends_on:
     - influxdb
  portainer:
    image: portainer/portainer
    restart: always
    volumes:
     - /var/run/docker.sock:/var/run/docker.sock
     - /srv/docker/portainer/data:/data
    ports:
     - 9000:9000
  nginx:
    image: nginx
    restart: always
    volumes:
     - /srv/docker/nginx/config/nginx.conf:/etc/nginx/nginx.conf
     - /srv/docker/nginx/ssl:/etc/nginx/ssl
     - /srv/docker/letsencrypt:/letsencrypt
    ports:
     - 8080:80
     - 443:443
```



<!--stackedit_data:
eyJoaXN0b3J5IjpbMTU4OTI5MzA1MSwtMTA0NDE0ODcwLDE0ND
cxNjE0MDcsOTA1MTcwMTcwLDMzNzI4NTMwOCwtODAwMTQ2Mjc0
XX0=
-->