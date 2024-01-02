# Homelab Setup Guide - Introduction

_Disclaimer: This guide is not meant to encourage or support piracy._

I've spent quite a bit of time over the last 12 months learning docker and linux, testing out different containers, and fine tuning a few to fit my needs. I don't consider myself a docker or linux pro by any means, I've made a few mistakes with my setup and I'm sure the ones I'll be sharing here could be better, but I wanted to create this guide for other people like me that enjoy spending time learning new things. This guide is primarily aimed at anyone that doesn't have an in-depth knowledge of both docker and linux, and will cover the steps needed to:

- Install & setup Ubuntu Server on a desktop/laptop
- Setup Docker and Docker-Compose
- Setup Portainer, Prowlarr, Sonarr, Radarr, Plex, Homarr, Watchtower, Tautulli, Overseerr, Speedtest Tracker, qBittorentVPN, Nginx Proxy Manager
- And finally, how to access some of these services outside of your home network

## Table of Contents

- [Homelab Setup Guide - Introduction](#homelab-setup-guide---introduction)
  - [Table of Contents](#table-of-contents)
  - [Hardware](#hardware)
  - [Installing Ubuntu Server](#installing-ubuntu-server)
  - [Directory Setup](#directory-setup)
 
# Hardware

Ubuntu Server is a very light weight OS, the recommended system requirements are:
```sh
    CPU: 1 gigahertz or better
    RAM: 1 gigabyte or more
    Disk: a minimum of 2.5 gigabytes
```
I could have gone with unraid or truenas, which would have worked just as well for my setup, but I had some previous experience with Ubuntu Server so it made sense for me to go that route. My server is running on a 5600G, 16GB, and I have two 4TD WD REDs for storage. It's a good idea to have a dedicate GPU for encoding, however, I've not had any issues with my current setup.

# Installing Ubuntu Server

Head over to https://ubuntu.com/download/server to download the Ubuntu Server ISO, then you'll need to either burn the ISO to a CD or create a bootable USB drive by using a program such as [rufus](https://rufus.ie/en/). You can dual boot with Windows, however, I will not be covering how to do that in this guide. If you choose to not dual boot, be aware that any data on your system may be lost. Once you've created the bootable drive, it's time to install Ubuntu Server on your system. 

With your system turned off, insert the USB drive and power on your system. If your system doesn't automatically boot from the USB, you will have to restart and enter the boot menu by rapidly pressing the ESC key, F2 key, F10 key, or the F12 key. Generally there will be an on-screen message that says which key is required to enter the boot menu at POST. Once the boot menu appears, select the USB drive and follow the prompts to complete installation. You can find an in-depth walkthrough [here](https://ubuntu.com/tutorials/install-ubuntu-server#1-overview), the main things I'd suggest paying attention to is the username and the server name you select, and make sure "Install OenSSH server" is selected. If you have multiple drives for your server, you'll also more than likely want to setup RAID. I went with RAID0 as nothing on my server is super important, but if you plan on storing important files on your server than you'll probably want to go with RAID5 or higher. Setting up RAID is out of scope for this guide, but I'll provide a walkthrough [here](https://support.us.ovhcloud.com/hc/en-us/articles/360006076940-How-to-Configure-Software-RAID-on-Ubuntu-18-04).

# Directory Setup

Once Unbuntu Server is installed and booted, you can either continue using the CLI or you can SSH to the server from another computer using command prompt(CMD) or PowerShell(PS) using the following command (`ssh your-username&serverIP`). Now it's time to setup some directories where your files will be stored. One thing to keep in mind when setting up directories is hard links and instant moves, which you can read up on at [trash guides](https://trash-guides.info/Hardlinks/Hardlinks-and-Instant-Moves/), but basically a hard link makes it possible to have a file(s) in multiple locations without taking up two, three, or more, times the amount of space the single file does.

Now we'll setup a directory for our movies/tv shows as well as directories for our containers. While at the CLI, enter the commands (`mkdir /mnt/data`), followed by (`mkdir ~/portainer prowlarr sonarr radarr plex homarr watchtower tautulli overseerr speedtest qbitvpn nginx`), and finally (`sudo ln -s $HOME/plex /mnt/data/`). This will create a symolic link between or plex directory and the directories where our files will be stored. Be aware, using sudo runs commands as root, or administrator, so be careful when using sudo. You can use the command (`ls -l`) to view the directories and information such as file system type and groups. Next we'll setup the directories where our downloads will go by using the commands (`cd /mnt/data`) and (`mkdir media torrents`). Now use (`cd media`) and then (`mkdir movies tv`).
