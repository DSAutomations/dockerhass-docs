


# Overview
Years ago as a new homeowner, I was starting my home automation journey. With an with an eye on the budget, I was drawn into the Home Assistant community. I found the do-it-yourself approach favorable to being locked into commercial cloud ecosystem. I opted to host HA on a Raspberry Pi 3b, on my Ubiquiti network consisting of an EdgeRouterX and Unifi access points. 

When I first deployed my setup, I avoided Hass.io because it was only in its infancy at that point, however the hassbian image seemed to be a reasonable option. Time went on, and I threw a few hours into development every week and I was always upgrading, adding on, and exploring new integrations. 

Ultimately, I've built some awesome features and automatons that my family enjoys, but the Raspberry pi has become a veritable house of cards. Along with homeassistant, I've installed all sorts of goodies for development, and integration. Nginx, pihole, mosquitto, apcupsd, certbot, awscli, samba, git, and the list goes on.

Like many others, I started seeing the message about python 3.5 depreciation recently, and through sweat and tears, managed to rebuild a new venv and get Homeassistant running in it without the warning. However, this came at the cost of a dramatic performance hit in the recorder component. I'm a big fan of graphs on my front end, and now they're all taking twice as long to load. Not sure where I went wrong, but one thing that I am sure about is that there is a better way, let's go...

I aim to give you a comprehensive guide for getting the *pictured* environment up and running from scratch, this environment includes the following:
* Homeassistant
* Nginx Front End 
* Mosquitto MQTT
* MariaDB & InfluxDB Back End
* Grafana
* Node Red
* Portainer

# Getting Started
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
 replace the `psk` line int the `wpa_supplicant` file with the key generated by this tool.

# Initial Config

At this point, I'm sitting at a nice fresh terminal on my pi. We need to start with a bit of housekeeping:
```
sudo apt update
sudo apt upgrade
```
Use the raspbian config script to set the hostname and password on your device:
```
sudo raspi-config
```
All of the following packagesare required to get Docker and docker-compose installed:
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
Next let's add a repository for us to pull the latest docker packages from, and we'll make it so that packages signed by the docker project are permitted to run.
```
echo "deb https://download.docker.com/linux/raspbian/ stretch stable" | sudo tee -a /etc/apt/sources.list
curl -fsSL https://yum.dockerproject.org/gpg | sudo apt-key add -
```
With all of this done, we should have everything we need to install docker. Let's go:
```
curl -fsSL get.docker.com -o get-docker.sh
sh get-docker.sh
```

```
sudo usermod -aG docker pi

# ~~ re-log now ~~

sudo pip install docker-compose
systemctl start docker.service
```

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE1NDk3MTY3NzRdfQ==
-->