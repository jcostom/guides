# Setup Homebridge on a Raspberry Pi

So, you've got all manner of home automation stuff going on, and most of it's HomeKit compatible, except for those couple of things, right? That's where [Homebridge](https://github.com/nfarina/homebridge) comes into play. Homebridge is a node.js server app that has a bunch of plugins for all sorts of platforms and devices. It's very small and lightweight, so it's well-suited for running on a Raspberry Pi. Natually, if you've got a Linux server at home already, much of this info will also apply there.

## Hardware Platform - Raspberry Pi Zero W

Sure, you could pick the Raspberry Pi 3, with it's 4 cores, 1G of RAM and WLAN, but now, with the Pi Zero W, you can have the power you need in a tiny package, and still have your WLAN.  The Pi Zero models offer 2 USB ports, 1 Micro USB for power input, and the other Micro USB as USB On The Go (aka OTG), and a Mini HDMI port for video output. This guide is designed for using the WLAN capabilities of the device to connect to your home network, so the only connection you'll end up needing is for power. I got a starter kit with all the toys (Pi, case, power, HDMI adapter, USB OTG adapter) for about $30 on [Amazon](http://a.co/6huaId0).

## Base OS - Raspbian Jessie

Raspbian is a Debian port to the Raspberry Pi family of systems. The "Jessie" release is essentially Debian 8. Head on over to the [Raspberry Pi website](https://www.raspberrypi.org/downloads/raspbian/) and grab the latest Raspbian Jessie Lite image. Got the image and unzipped it? Write it out to a Micro SD card and you're ready to go. On a PC, use something like [Rufus](https://rufus.akeo.ie) for this job. On a Mac, just crack open a terminal window and (assuming the SD is /dev/disk2):

```
diskutil umount /dev/disk2s1
dd if=/path/to/image.img of=/dev/rdisk2 bs=1m
```

So, why `/dev/rdisk2` instead of `/dev/disk2`? Simple - time to write the image.  The raw device node will be *much* faster than the regular device node.

Image written out? Great, put it in the Pi, assemble the case, connect a display & keyboard and boot the system. The system will start, auto-resize the root filesystem to take up the rest of the memory card's space and reboot. Once booted, you'll have a few jobs to do before you'll be properly ready to roll.

Default login is `pi`, with password `raspberry`. You should change this password at once.

You should also enable SSH immediately. This is done in `raspi-config`.  Look under `Interfacing Options > SSH` to enable the SSH server. The `raspi-config` utility usually asks for a reboot after changes. I know, it's a lot of rebooting, but at least it boots quickly, right?

### Configure Keyboard Layout

I'm assuming you're using a US keyboard here. Raspbian assumes you're using a GB keyboard. It's an easy fix.  Change the /etc/default/keyboard file (you'll need to use sudo to edit the file as root) to change the `XKBLAYOUT` setting to "us".  Reboot after you make this change.

### Configure WLAN

Configuring the WLAN is very straight forward.  `sudo vi /etc/wpa_supplicant/wpa_supplicant.conf` (or nano instead of vi if you'd rather). You'll need to set the WLAN regulatory domain as well (what country you're in), again, I'm assuming the US. Mine looks like this (assumes WPA2, PSK with AES):

```
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
network={
	ssid="<myssid>"
	psk="<mypsk>"
	proto=RSN
	key_mgmt=WPA-PSK
	pairwise=CCMP
	auth_alg=OPEN
}
```
Reboot after you've made these changes. When the Pi comes back online, you should be on the network!

### Update the OS, Install Node

Next, you should update your packages in the usual (for Debian) manner.
```
sudo apt-get update
sudo apt-get upgrade
```
If you replaced the kernel, or just want to be safe/sure, reboot after the update is completed.

The Pi Zero W is essentially the original Pi Model B, miniaturized. Sadly, we must install Node manually on this platform.  Fret not, it's not that hard.  Visit [nodejs.org](https://nodejs.org/en/download/) and grab the latest LTS build for ARMv6 (current LTS release is 6.11 as I write this).

To install, unpack as follows:

```
cd /usr/local
sudo tar xvf /path/to/node-v6.11.0-linux-armv6l.tar.xz --strip=1
```
That last flag `--strip=1` tells tar to ignore one level of path in the archive, so your unpacked files neatly fall into /usr/local, ready to go.

If you went against my advice of the Pi Zero W in favor of the Pi 3, it (slightly) paid off for you - installing Node is slightly easier, since for the newer Pi CPUs, there are pre-packaged debs.  If that's the case, per the nodes at [nodejs.org](https://nodejs.org/en/download/package-manager/), just do this:

```
curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
sudo apt-get install -y nodejs
```
If you've got the official node packages from Raspbian (which are old) installed, you'll want to get rid of those packages: `sudo apt-get remove --purge npm node nodejs`

Lastly, you will want to update the npm package manager tool. `sudo npm i -g npm` will make that happen. After that's done, you'll want to do a `hash -r` to refresh your search path.

## Install Homebridge

You've arrived at last. Time to install the thing we're looking for to get the job done!

### Support packages

Don't worry, it's only a couple of packages that get installed.

```
sudo apt-get install libavahi-compat-libdnssd-dev
```

### Homebridge and modules

Installing homebridge is quite straight forward.  All it takes is `sudo npm i -g --unsafe-permissions homebridge`, and some patience while you wait for the software to build and install.

Ok, great. You installed it. Unfortunately, it's totally useless with out 2 more things - modules for your devices and a configuration. We will use the homebridge-platform-myq and homebridge-harmonyhub modules as the great example.

Install the modules with `sudo npm i -g homebridge-platform-myq` and `sudo npm i -g homebridge-harmonyhub`.

Configuration syntax is relatively simple.  I'll assume you plan on running this as the `pi` user. Default config location is `~/.homebridge/config.json`. I see no reason to deviate from this.

Here's a working config example:

```
{
    "bridge": {
        "name": "Jenny's Homebridge",
        "username": "DE:AD:BE:EF:CA:FE",
		"manufacturer": "Homebridge",
		"model": "Pi Zero W",
		"serialNumber": "8675309",
        "pin": "123-45-678"
    },
    "description": "Raspberry Pi Zero W with homebridge to extend HomeKit to various devices",
    "platforms": [
        {
            "platform": "MyQ",
            "name": "MyQ",
            "user": "<yourusername>",
            "pass": "<yourpassword>",
            "brand": "Chamberlain"
        },
        {
            "platform": "HarmonyHub",
            "name": "Living Room Harmony"
        }
    ],
    "accessories": [
    ]
}
```

Last bit - setting up homebridge to automatically start at boot.

Create the following file as /etc/systemd/system/homebridge.service:

```
[Unit]
Description=Node.js HomeKit Server
After=syslog.target network-online.target

[Service]
Type=simple
User=pi
ExecStart=/usr/local/bin/homebridge
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
```

To finish up the job, just a few commands are needed:

```
sudo systemctl daemon-reload
sudo systemctl enable homebridge
sudo systemctl start homebridge
```

You can check the status with `service homebridge status`.

### Add to HomeKit on your iPhone

Go into the Home app, and add the device. You'll need to type the PIN manually, but as above, our example is 123-45-678. Feel free to change yours as you like before starting up.
