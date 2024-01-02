# Homelab Setup Guide - Introduction

_Disclaimer: This guide is not meant to encourage or support piracy._

I've spent quite a bit of time over the last 12 months learning docker and linux, testing out different containers, and fine tuning a few to fit my needs. I don't consider myself a docker or linux pro by any means, I've made a few mistakes with my setup and I'm sure the ones I'll be sharing here could be better, but I wanted to create this guide for people like me that enjoy learning through trial and error. This guide is primarily aimed at anyone that doesn't have an in-depth knowledge of both docker and linux, and will cover the steps needed to:

- Install & setup Ubuntu Server on a desktop/laptop
- Setup Docker and Docker-Compose
- Setup Portainer, Prowlarr, Sonarr, Radarr, Plex, Homarr, Watchtower, Tautulli, Overseerr, Speedtest Tracker, qBittorentVPN, Nginx Proxy Manager
- And finally, how to access some of these services outside of your home network

## Table of Contents

- [Homelab Setup Guide - Introduction](#homelab-setup-guide---introduction)
  - [Table of Contents](#table-of-contents)
  - [Hardware](#hardware)
  - [Installing Ubuntu Server](#installing-ubuntu-server)
  - [Server Network](#server-network)
  - [Firewall](#firewall)
  - [Directory Setup](#directory-setup)
  - [Install Docker and Docker-Compose](#install-docker-and-docker-compose)
  - [Setup Portainer](#setup-portainer)
 
# Hardware

Ubuntu Server is a very light weight OS, the recommended system requirements are:
```sh
    CPU: 1 gigahertz or better
    RAM: 1 gigabyte or more
    Disk: a minimum of 2.5 gigabytes
```
I could have gone with unraid or truenas, which would have worked just as well for my setup, but I had some previous experience with Ubuntu Server so it made sense for me to go that route. My server is running on a 5600G, 16GB, and I have two 4TD WD REDs for storage. It's a good idea to have a dedicate GPU for transcoding, however, I've not had any issues with my current setup.

# Installing Ubuntu Server

Head over to https://ubuntu.com/download/server to download the Ubuntu Server ISO, then you'll need to either burn the ISO to a CD or create a bootable USB drive by using a program such as [rufus](https://rufus.ie/en/). You can dual boot with Windows, however, I will not be covering how to do that in this guide. If you choose to not dual boot, be aware that any data on your system may be lost. Once you've created the bootable drive, it's time to install Ubuntu Server on your system. 

With your system turned off, insert the USB drive and power on your system. If your system doesn't automatically boot from the USB, you will have to restart and enter the boot menu by rapidly pressing the ESC key, F2 key, F10 key, or the F12 key. Generally there will be an on-screen message that says which key is required to enter the boot menu at POST. Once the boot menu appears, select the USB drive and follow the prompts to complete installation. You can find an in-depth walkthrough [here](https://ubuntu.com/tutorials/install-ubuntu-server#1-overview), the main things I'd suggest paying attention to is the username and the server name you select, and make sure "Install OpenSSH server" is selected. If you have multiple drives for your server, you'll also more than likely want to setup RAID. I went with RAID0 as nothing on my server is super important, but if you plan on storing important files on your server then you'll probably want to go with RAID5 or higher. Setting up RAID is out of scope for this guide, but I'll provide a walkthrough [here](https://support.us.ovhcloud.com/hc/en-us/articles/360006076940-How-to-Configure-Software-RAID-on-Ubuntu-18-04).

# Server Network

While not entirely necessary, I've gone ahead and given my server a static IP. Most routers will allow you to do it in the gui, however, I've set it up on my server. Follow the instructions below if you'd like to do the same. First, you need to identify the available interfaces by using the command `ip a`, make note of the interface name and subnet mask your network uses. Now create the netplan config using the command `sudo nano /etc/netplan/99_config.yaml`. I've provided my config below as an example, but you will need to change things to suite your network:
```sh
network:
  renderer: networkd
  ethernets:
    enp42s0:
      addresses:
        - 192.168.X.X/XX
      nameservers:
        addresses: [4.2.2.2, 8.8.8.8]
      routes:
        - to: default
          via: 192.168.X.X
  version: 2
```

# Firewall

_What is UFW?_

> Uncomplicated Firewall (ufw) is a frontend for iptables and is particularly well-suited for host-based firewalls. ufw provides a framework for managing netfilter, as well as a command-line interface for manipulating the firewall. ufw aims to provide an easy to use interface for people unfamiliar with firewall concepts, while at the same time simplifies complicated iptables commands to help an administrator who knows what he or she is doing. ufw is an upstream for other distributions and graphical frontends.

To start, you should enable some basic things such as SSH and logging by using the following commands:
```sh
sudo ufw allow ssh/tcp
sudo ufw logging on
sudo ufw enable
sudo ufw status
```
We will need to allow more ports later in this guide, but this should be fine for now.

# Directory Setup

Once Ubuntu Server is installed and booted, you can either continue using the CLI or you can SSH to the server from another computer using command prompt(CMD) or PowerShell(PS) using the following command `ssh yourusername@yourserverIP`. Now it's time to setup some directories where your files will be stored. One thing to keep in mind when setting up directories is hard links and instant moves, which you can read up on at [trash guides](https://trash-guides.info/Hardlinks/Hardlinks-and-Instant-Moves/), but basically a hard link makes it possible to have a file or files in multiple locations without taking up two, three, or more times the amount of space the single file does.

Now we'll setup a directory for our movies & tv shows files. While at the CLI, enter the commands `mkdir /mnt/data`, this is the directory where our files for movies and tv shows will be stored. Next, switch to the data directory and create the directories we need by using the commands `cd /mnt/data` and `mkdir media torrents`, now we'll create the directories for movies and tv by using the commands `cd /mnt/data/media` and `mkdir movies tv`. Now all we need to do is change the permissions by using the commands `sudo chown -R $USER:$USER /data` and `sudo chmod -R a=,a+rX,u+w,g+w /data`.

Next we'll get our container directory setup by using the command `cd` to get back to your home directory, followed by `mkdir containers`. and finally `sudo ln -s $HOME/containers /mnt/data/`. This will create a symolic link between our containers directory and the directories where our files will be stored. Be aware, using `sudo` runs commands as root, or administrator, so be careful when using sudo. We're only going to setup one directory in containers for Portainer, so use the commands `cd containers` and `mkdir portainer/data`.

# Install Docker and Docker-Compose

The official instructions can be found [here]([https://trash-guides.info/Hardlinks/Hardlinks-and-Instant-Moves/](https://docs.docker.com/engine/install/ubuntu/#install-docker-ce-1)https://docs.docker.com/engine/install/ubuntu/#install-docker-ce-1), but I'll be including everything you should need in this guide. We'll start by removing any conflicting packages by running the command: 
```sh
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```
Next we'll setup Docker's apt repository:
```sh
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
Now run the following to install the lastest version:
```sh
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
And verify the Docker Engine is installed:
```sh
sudo docker run hello-world
```
Now update the package and install the latest version of Docker Compose:
```sh
sudo apt-get update
sudo apt-get install docker-compose-plugin
```
And verify it has been installed correctly:
```sh
docker compose version
```

# Setup Portainer

_What is Portainer?_

> "Portainer is a powerful, open source toolset that allows you to easily build and manage containers in Docker, Docker Swarm, Kubernetes and Azure ACI. Portainer hides the complexity of managing containers behind an easy-to-use UI. By removing the need to use the CLI, write YAML or understand manifests, Portainer makes deploying apps and troubleshooting problems so easy that anyone can do it.

When I first started my homelab server I was using one large yml file with a bunch of containers nested inside, I found it started to become cumbersome to modify so I wanted something to make the process of managing containers even simpler than a yml file. That's when I found Portainer, I find it makes the process of setting up, modifying, and deleting containers very easy. As their description says, it really does make it so easy anyone can do it.

Docker-compose uses a yml(yet another markup language) file to create and maintain docker containers, to create the yml file for Portainer change to the `portainer` directory. If you're at home, you can use the command `cd containers/portainer/`. Next, use the command `sudo nano docker-compose.yml` to create the yml. We'll be setting up Portainer CE, you can find the official install documentary [here](https://docs.portainer.io/start/install-ce). I've provided an example of my docker-compose.yml with the portainer entry in it, I've also commented out some explanations:
```sh
version: "3"
networks:
  default:
    driver: bridge #bridging the container network to my home network
services:
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:latest
    networks:
      - default #setting the network for portainer to bridge
    ports:
      - 9443:9443 #left port is the host port, right is the container port
    volumes:
      - /home/your-username-here/portainer/data:/data #left is the host directory, right is the directory on the container
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
```
Now we're ready to start your first container, use the command `docker-compose up -d` and once it shows that the container is up you can log into the web ui by going to `http://serverIP:9443`.
