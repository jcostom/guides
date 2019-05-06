# Containerized Homebridge on Ubuntu

## Base OS Installation

Folks, I'm not about to walk you through the basics of install Ubuntu Linux. There are dozens, probably hundreds, perhaps even thousands of such writeups. I lack the motivation to reinvent that wheel yet again. :-) I'll recommend Ubuntu's latest LTS release (18.04 as of when I'm writing this, but YMMV, so carry on as you see fit).

## Install Docker CE

The Docker folks have a [very simple guide](https://docs.docker.com/install/linux/docker-ce/ubuntu/) to do this. You should follow it. Really.

When you're done, I'd consider adding your own userid to the docker group.

`sudo usermod -aG docker <yourUserIDgoesHere>`

Logout and back in, and whammo, you're able to execute Docker commands without the need of using sudo.

## Install the Homebridge Container

Start by pulling the latest copy of the `oznu/homebridge` container by doing:

`docker pull oznu/homebridge`

## Figure Out Some Parameters

You'll need to know the following:

* UID the container runs as (`PUID` below)
* GID the container runs as (`PGID` below)
* The timezone you'll run as (`TZ` below)
* The directory on the host where you'll store the configuration files (remember, a container is ephemeral, and doesn't preserve data if you delete and recreate the container).

## Instantiate Your Homebridge Container

Read the info [published about the container](https://hub.docker.com/r/oznu/homebridge) by the creator to get started. There's plenty of good info in there about optional flags (ie -e parameters), and determining what UID and GID you'd like to run as.

Example command to create a Homebridge instance using the following parameters:

* UID / GID both 1001
* America/New_York timezone
* Config data stored in /home/homebridge on the host, maps to /homebridge *inside* the container.

```
docker run -d \
    --restart=unless-stopped \
    --net=host \
    --name=homebridge \
    -e TZ=America/New_York \
    -e PUID=1001 -e PGID=1001 \
    -e HOMEBRIDGE_CONFIG_UI=1 \
    -e HOMEBRIDGE_CONFIG_UI_PORT=8989 \
    -v /home/homebridge:/homebridge \
    oznu/homebridge
```
