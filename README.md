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
  - [Prowlarr](#prowlarr)
  - [Sonarr](#sonarr)
  - [Radarr](#radarr)
  - [qBittorrentVPN](#qbittorrentvpn)
  - [Arr Settings](#arr-settings)
    - [Prowlarr Settings](#prowlarr-settings)
    - [Sonarr Settings](#sonarr-settings)
    - [Radarr Settings](#radarr-settings)
  - [Plex](#plex)
  - [Homarr](#homarr)
 
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
If you find you're unable to connect to any of the containers setup in this guide, make sure to add it to ufw by using `sudo ufw allow <required port>`.

# Directory Setup

Once Ubuntu Server is installed and booted, you can either continue using the CLI or you can SSH to the server from another computer using command prompt(CMD) or PowerShell(PS) using the following command `ssh yourusername@yourserverIP`. Now it's time to setup some directories where your files will be stored. One thing to keep in mind when setting up directories is hard links and instant moves, which you can read up on at [trash guides](https://trash-guides.info/Hardlinks/Hardlinks-and-Instant-Moves/), but basically a hard link makes it possible to have a file or files in multiple locations without taking up two, three, or more times the amount of space the single file does.

Now we'll setup a directory for our movies & tv shows files. While at the CLI, enter the commands `mkdir /mnt/data`, this is the directory where our files for movies and tv shows will be stored. Next, switch to the data directory and create the directories we need by using the commands `cd /mnt/data` and `mkdir media torrents`, now we'll create the directories for movies and tv by using the commands `cd /mnt/data/media` and `mkdir movies tv`. Now all we need to do is change the permissions by using the commands `sudo chown -R $USER:$USER /data` and `sudo chmod -R a=,a+rX,u+w,g+w /data`.

Next we'll get our container directory setup by using the command `cd` to get back to your home directory, followed by `mkdir containers`. and finally `sudo ln -s $HOME/containers /mnt/data/`. This will create a symolic link between our containers directory and the directories where our files will be stored. Be aware, using `sudo` runs commands as root, or administrator, so be careful when using sudo. First we'll setup a directory for Portainer, so use the commands `cd containers` and `mkdir portainer/data`. Next, we'll create the directories for the rest of the containers using the command `mkdir prowlarr sonarr radarr plex homarr watchtower tautulli overseerr speedtest qbitvpn nginx`

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

Docker-compose uses a yml(yet another markup language) file to create and maintain docker containers, to create the yml file for Portainer change to the `portainer` directory. If you're at home, you can use the command `cd containers/portainer/`. Next, use the command `sudo nano docker-compose.yml` to create the yml. We'll be setting up Portainer CE, you can find the official install document [here](https://docs.portainer.io/start/install-ce). I've provided an example of my docker-compose.yml with the portainer entry in it, I've also commented out some explanations:
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
Now we're ready to start your first container, use the command `docker-compose up -d` and once it shows that the container is up, enter `http://serverIP:9443` in your browser of choice to begin the [setup](https://docs.portainer.io/start/install-ce/server/setup) process. Once you've setup an admin account and you've created the local environment, click on "local" and you should see something similar to this:

![Portainer](https://github.com/SupraTHICC/homelab-setup/assets/92880114/ccd85f13-070c-465f-b1f5-1eb50f7421fb)

Once you're at the dashboard, click on stacks and in the top right click "Add stack". The first container we're going to create is Prowlarr.

# Prowlarr

_What is Prowlarr?_

> [Prowlarr](https://docs.linuxserver.io/images/docker-prowlarr/) is a indexer manager/proxy built on the popular arr .net/reactjs base stack to integrate with your various PVR apps. Prowlarr supports both Torrent Trackers and Usenet Indexers. It integrates seamlessly with Sonarr, Radarr, Lidarr, and Readarr offering complete management of your indexers with no per app Indexer setup required (we do it all).

Now that you're at the add stack screen and you're on web editor, give your stack a name and enter the following:
```sh
services:
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC #Change this to your TZ
    volumes:
      - /home/your-username-here/containers/prowlarr:/config 
    ports:
      - 9696:9696
    restart: unless-stopped
```
[Note:](https://docs.linuxserver.io/general/understanding-puid-and-pgid/) `Using the PUID and PGID allows our containers to map the container's internal user to a user on the host machine.`
TZ identifiers can be found [here](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List)

Once you've added your information and you're ready to start the container, click on "Deploy the stack". We need to now setup our indexers in Prowlarr, you can find a more in-depth guide [here](https://wiki.servarr.com/prowlarr/quick-start-guide), to do that go to `http://serverIP:9696` and you should see something similar to this:

![prowlarr](https://github.com/SupraTHICC/homelab-setup/assets/92880114/c86857ed-010b-4a47-9dba-0128db244726)

On the add indexer screen, you can either search for an indexer by name or you can scroll through the list. I don't have a lot of indexers, but I mainly use 1337x, LimeTorrents, and TorrentFunk. After you've selected an indexer, select one of the URLs in base URL, click test and if it comes back with a green check, click save. Once we've setup Radarr, Sonarr and qBittorrent, we'll come back to Prowlarr to add them.

# Sonarr

_What is Sonarr?_

> [Sonarr](https://docs.linuxserver.io/images/docker-sonarr/) (formerly NZBdrone) is a PVR for usenet and bittorrent users. It can monitor multiple RSS feeds for new episodes of your favorite shows and will grab, sort and rename them. It can also be configured to automatically upgrade the quality of files already downloaded when a better quality format becomes available.

It's time to add Sonarr. Head back to the Portainer UI, click stacks, and add stack. Name this one Sonarr and you can use the example below, modifying it if needed:
```sh
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
        - PUID=1000
        - PGID=1000
        - TZ=America/New_York #Change to your TZ
    volumes:
        - /home/your-username-here/containers/sonarr:/config
        - /mnt/data:/data
    ports:
        - 8989:8989
    restart: unless-stopped
```
Click "Deploy the stack" and go to `http://serverIP:8989` to begin the setup. When you're at the Sonarr UI, click settings on the left side and then click media management. In media management click add root folder and select the folder `/data/media/tv`. We'll come back to Sonarr once Radarr and qBitorrent are setup.

# Radarr

_What is Radarr?_

> [Radarr](https://github.com/Radarr/Radarr) is a movie collection manager for Usenet and BitTorrent users. It can monitor multiple RSS feeds for new movies and will interface with clients and indexers to grab, sort, and rename them. It can also be configured to automatically upgrade the quality of existing files in the library when a better quality format becomes available.

Once again, in the Portainer UI, add a stack, name this one Radarr and you can use the example below, modifying it if needed:
```sh
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    hostname: radarr
    environment:
        - PUID=1000
        - PGID=1000
        - TZ=America/New_York
    volumes:
        - /home/your-username-here/containers/prowlarr:/config
        - /mnt/data:/data
    ports:
        - 7878:7878
    restart: unless-stopped
```
Click "Deploy the stack" and go to `http://serverIP:7878` to begin the setup. When you're at the Radarr UI, click settings on the left side and then click media management. In media management click add root folder and select the folder `/data/media/movies`. We'll come back to Radarr qBitorrent are setup.

# qBittorrentVPN

_What is qBittorentVPN?_

> [qBittorrent](https://github.com/qbittorrent/qBittorrent) is a bittorrent client programmed in C++ / Qt that uses libtorrent (sometimes called libtorrent-rasterbar) by Arvid Norberg. It aims to be a good alternative to all other bittorrent clients out there. qBittorrent is fast, stable and provides unicode support as well as many features.
> The [OpenVPN](https://openvpn.net/faq/what-is-openvpn/) Community Edition (CE) is an open source Virtual Private Network (VPN) project. It creates secure connections over the Internet using a custom security protocol that utilizes SSL/TLS. 

The qBittorrentVPN docker hub can be found [here](https://hub.docker.com/r/dyonr/qbittorrentvpn/) which provides a lot of information and the environment variables that can be used. For this container you will need to have a VPN, if you choose to go with OpenVPN like I have you can find a list of supported VPN providers [here](https://haugene.github.io/docker-transmission-openvpn/supported-providers/). I use NordVPN, I know there are _better_ alternatives available but it's just what I'm used to, as well as OpenVPN. 

In the Portainer UI, click add a stack and name this one qbitvpn. Below is an example of my settings which you will need to modify with your VPN providers info:
```sh
services:
  qbitvpn:
    image: dyonr/qbittorrentvpn:latest
    container_name: qbitvpn
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun
    environment:
      - VPN_ENABLED=yes
      - VPN_TYPE=openvpn
      - VPN_USERNAME=redacted
      - VPN_PASSWORD=redacted
      - NAME_SERVERS=1.1.1.1,1.0.0.1
      - PUID=1000
      - PGID=1000
      - HEALTH_CHECK_HOST=www.google.com
      - HEALTH_CHECK_INTERVAL=300
      - HEALTH_CHECK_AMOUNT=10
      - RESTART_CONTAINER=no
      - LAN_NETWORK=192.168.X.X/XX
    volumes:
      - /home/your-username-here/containers/qbitvpn:/config
      - /mnt/data/torrents:/data/torrents
    ports:
      - 8112:8080 #I've changed the host port to 8112 instead of 8080 as another container is already using it
      - 8999:8999
      - 8999:8999/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```
I would recommend reading the qBittorrentVPN docker hub before deploying the stack, once you've done that deploy it and go to `http://serverIP:8112`(or whichever port you used) and enter the default credentials, which are `username: admin password: adminadmin`. Click on settings, then click the "webui" tab and change the default password to your own. Next, click on the "downloads" tab and your settings should look similar to this:

![qbit](https://github.com/SupraTHICC/homelab-setup/assets/92880114/986aaf1f-93ee-41ae-9951-c9381f208302)

Next we'll be going back to Prowlarr, Sonarr, and Radarr to finalize the settings. 

# Arr Settings

## Prowlarr Settings

Now that we've got our main containers set up, we need to go back to add some things that we couldn't add before. First, go to Prowlarr and click settings then apps. Once there, click the + to add an app and select Radarr. Fill in the fields so they look similar to this:

![prowlarr-apps](https://github.com/SupraTHICC/homelab-setup/assets/92880114/4352472a-96d6-4259-ac52-0e207fc1ba82)

In the Prowlarr server and Radarr section, enter `http://serverIP:9696` for Prowlarr and `http://serverIP:7878` for Radarr as well as the API key in Radarr that can be found in settings>general, click test and the if you get a green check click save, you can then do the same to add Sonarr. Next, click download clients, click the + and select qBittorent(_I don't think a download client is needed in Prowlarr, but it's good practice_). Fill in the fields so they look similar to this:

![prowlarr-download](https://github.com/SupraTHICC/homelab-setup/assets/92880114/a97bbce1-1652-46bf-887b-c0ba0ebf264a)

Again, click test and if you get a green check, click save. Now click general and fill in the fields so they look similar to this:

![prowlarr-host](https://github.com/SupraTHICC/homelab-setup/assets/92880114/1c9bd307-d638-4f9c-a834-13d36555abc1)

That's about it for Prowlarr, now we can get Sonarr configured. 

## Sonarr Settings

Once you're at the Sonarr UI, click settings and click indexers. If you setup any indexers in Prowlarr, they should be populated here. Next, click download clients and enter the same information you entered in Prowlarr. And finally, click general and fill out the information the same as you did for Prowlarr.

## Radarr Settings

Just like Sonarr, you should fill out the same information as you did previously. Make sure to change anything that's specific to each container, such as port, if need be. Now that you have the indexers, apps, and download client set up, you should test a download to make sure everything is working. In Sonarr/Radarr, click Add new and search for a show or movie. These are the settings I typically use:

![sonarr](https://github.com/SupraTHICC/homelab-setup/assets/92880114/5a0176a3-59db-44f8-ba41-f77c1c592ecf)

Once you click add, go to qBittorrent and make sure it downloads. If you receive an error, SSH into your server and in your qbitvpn/qBittorent/data directory there should be a log which may help resolve the error. 

# Plex

_What is Plex_?

> [Plex](https://hub.docker.com/r/linuxserver/plex) organizes video, music and photos from personal media libraries and streams them to smart TVs, streaming boxes and mobile devices. This container is packaged as a standalone Plex Media Server. Straightforward design and bulk actions mean getting things done faster.

Now we'll get our Plex container set up. Head back to Portainer, add a stack, name it Plex, and you can use the example below as a starting point:
```sh
services:
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    network_mode: host
    volumes:
    - /home/your-username-here/containers/plex:/config
    - /mnt/data/media:/media
    environment:
        - PUID=1000
        - PGID=1000
        - version=docker
    ports:
        - 32400:32400
    restart: unless-stopped
```
Deploy the stack but this time instead of going to Plex in a browser, open CMD/PS and enter the following `ssh -L32400:localhost:32400 your-username@serverIP`. This will create a tunnel into your server, and you can now configure Plex. Once you're at `http://serverIP:32400` you'll need to create an account, if you don't currently have one, and then give your server a name once you've claimed it. When adding your libaries, make sure to point the movies library to `/media/movies` and the TV library to `/media/tv`. I haven't change much settings wise in Plex, other than allowing others access to my media. A couple of settings you should look at is "scan my library automatically" and remote access, other than that I usually leave all the settings on default. 

# Homarr

_What is Homarr?_

> [Homarr](https://github.com/ajnart/homarr) - a sleek, modern dashboard that puts all of your apps and services at your fingertips. With Homarr, you can access and control everything in one convenient location. Homarr seamlessly integrates with the apps you've added, providing you with valuable information and giving you complete control.

