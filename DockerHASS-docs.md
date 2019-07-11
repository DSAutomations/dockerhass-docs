


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
We're starting from the ground up. I'm using a Raspberry Pi 3b. Let's grab the latest [Raspbian Strech Lite](https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2019-04-09/) and flash it to an SD card using an app like [etcher](https://www.balena.io/etcher/). 

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
                      python-pip -y
```
Next let's add a repository for us to pull the latest docker packages from, and alos, we'll make it so that packages signed by the docker project are permitted to run.
```
echo "deb https://download.docker.com/linux/raspbian/ stretch stable" | sudo tee -a /etc/apt/sources.list
curl -fsSL https://yum.dockerproject.org/gpg | sudo apt-key add -
```
With all of this done, we should have everything we need to install docker. This might take a little while, let's go:
```
curl -fsSL get.docker.com -o get-docker.sh
sh get-docker.sh
```
Now that Docker is installed we'll make things a bit easier by putting the pi user in the docker group:
```
sudo usermod -aG docker pi
```
After doing this, you must **log out and back in** for the group permissions to be applied.

Almost done, let's install docker-compose:
```
sudo pip install docker-compose
```
## Docker Images
Images are at the core of docker, we're going to use a set of them to create our environment. You can create your own, but that is beyond the scope of this guide. For today, we're going to use public images from [Docker Hub](https://hub.docker.com). 

The first time we launch our stack, the images will be downloaded and cached on our system. Subsequent runs will utilize these downloaded images.

One important note is that not all images available on Docker Hub will be compatible with the Raspberry Pi. An image needs to be created with a compatible CPU architecture. Here are the images we will be using. I've found these to be reasonably popular images which are compatible with my Raspberry Pi 3b:

* [homeassistant/raspberrypi3-homeassistant](https://hub.docker.com/r/homeassistant/raspberrypi3-homeassistant)
* [jsurf/rpi-mariadb](https://hub.docker.com/r/jsurf/rpi-mariadb)
* [nodered/node-red-docker](https://hub.docker.com/r/nodered/node-red-docker)
* [influxdb](https://hub.docker.com//influxdb)
* [fg2it/grafana-armhf](https://hub.docker.com/r/fg2it/grafana-armhf)
* [portainer/portainer](https://hub.docker.com/r/portainer/portainer)
* [nginx](https://hub.docker.com//nginx)

### Tags
To get the desired version of a container, you may need to append a tag to it. The standard syntax is *`<ImageName>:<Tag>`.*  Here is the list of both images *and* tags that we will deploy: 
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
mkdir /srv/docker/letsencrypt/live
mkdir /srv/docker/mariadb
mkdir /srv/docker/nodered
mkdir /srv/docker/influxdb
mkdir /srv/docker/grafana
mkdir /srv/docker/portainer
mkdir /srv/docker/nginx/config
mkdir /srv/docker/nginx/ssl
```
## Optional: Setup Samba
Of course, you can do all of your config file creation and editing at the command line, however this can be a bit cumbersome


If you're like me and you keep your Homeassistant config on GitHub, now would be the time to clone your repository:
```
sudo apt install git
git clone https://github.com/DSAutomations/HomeAssistantConfig.git /srv/docker/homeassistant
```
Otherwise, copy your config into the homeassistant directory we just created. You can use SCP over command line or you can use samba. If you're brand new to homeassistant and don't yet have a config, create a file called `configuration.yaml` in `/srv/docker/homeassistant` and add the following to get you started:
```
homeassistant:
	default_config:
```



     - /srv/docker/HomeAssistantConfig:/config
     - /srv/docker/letsencrypt/live:/letsencrypt
     - /srv/docker/mariadb/config:/etc/mysql/conf.d
     - /srv/docker/mariadb/data:/var/lib/mysql
     - /srv/docker/nodered:/data
     - /srv/docker/influxdb/data:/var/lib/influxdb
     - /srv/docker/grafana/data:/var/lib/grafana
     - /srv/docker/portainer/data:/data
     - /srv/docker/nginx/config/nginx.conf:/etc/nginx/nginx.conf
     - /srv/docker/nginx/ssl:/etc/nginx/ssl
     - /srv/docker/letsencrypt:/letsencrypt

```
version: '3'
services:
  homeassistant:
    container_name: home-assistant
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

# Setup Samba
To make things a bit easier during initial configuration, you may want to setup a samba share to access your docker volumes. We'll only want to run this while we're actively working on the system.  To do this, create a new directory in your home folder called samba:
```
mkdir ~/samba
```
Change to the directory and create a `docker-compose.yml` file within:
```
cd ~/samba
nano docker-compose.yml
```
paste the following into this file:
```
```
Then press `Ctrl-X` then `y` then `enter` to save the file.

Make sure that your working directory is still `~/samba` and run the following to start the container:
```
docker-compose up -d
```

<!--stackedit_data:
eyJoaXN0b3J5IjpbOTU0MDc0ODE2LC03NDE2Mzc1NDksLTQyNj
Y0MzEyMCwtMzk5OTQzODY2LC0xMjc5NDk5MzUxLC0xMzY4ODU2
ODY0LDQ1OTA5ODA0MSwtMTg2MTMwMjc2MiwtMTgwNTQ2ODk1Ny
w1MzUzMzQ3NDQsLTM3MDgzNDI0NSwtMTA0ODE3OTI3NiwxODUw
MjYwNjczLC00OTMxNTYzOTAsNjU4MTAyNDcwLC0xMjQ4MjM0Nj
c0LDE0NTY3MTgwNzEsLTE2Mzc5MjI2NTIsMTU3Njk0NTE0Miwt
MTMzNDQ2MzA4NV19
-->