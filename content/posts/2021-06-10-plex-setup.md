---
title: Automating Plex and avoiding your ISP with Docker!
startDate: 2021-06-10 18:00:00
date: 2021-08-16 03:37:00
categories:
    - homelab, tutorial
tags:
    - plex, docker, vpn, automation
keywords:
    - homelab, tutorial, plex, docker, vpn, sonarr, radarr
---
Alright, so I finally got around to automating my Plex server so here's a little write-up detailing how I did it for anyone trying to acheive the same (and also for me when I inevitably break everything XD)!

## Preface
This guide assumes you already have a decent server, a working Plex installation (if not that's ok as well), media locations mounted on your system, and a VPN provider. If you're starting completely from scratch, I highly reccommend reading through [this guide](https://blog.muffn.io/unlimited-plex-storage-via-google-drive-and-rclone/) by MonsterMuffin as it's what I used get this project on the road. That guide uses GDrive and Ubnutu and I'll be doing the same but everything should still work with minimal changes if your setup is different.

Here's a diagram of what we'll be setting up:
![image](/images/2021-06-10-plex-setup/plex_setup_flowchart.png)
Don't worry if it's confusing now because I'll be going over all of it soon!

## Why did you make this and what all does it aim to do?
Basically I wanted to completely automate my plex server so that I don't have to spend time manually finding, renaming, and importing files. I also wanted the peace of mind that if I ever mess up, I'll be able to quickly fix things. Docker has been HUGE for this!
Also, I wanted to hide my 😉**completely legal**😉 downloads from my ISP while not slowing down the rest of my stack. Now they won't judge my taste in linux distros!

## The Setup
Aight let's get to it!
### Download files
Firstly, you're going to want to clone my `plex-setup` repo as follows:
```bash
cd ~
git clone git@github.com:switchswap/plex-setup.git
```
This will clone my setup repository to your home directory.
You can ignore most of the files in there apart from the `docker` directory since this guide is only about that part.

### Install docker
Next, you'll want to install docker on your system. Use your favorite package manager to do that I suppose! 

Just make sure you only have one docker install though! I forgot I had the snap package installed for it and that caused me some trouble with ghost containers and blocked ports before I uninstalled it.

### Edit configs
Navigate to the docker folder with `cd ~/plex-setup/docker/` and edit the `.env` file to your preference.

You'll need to edit the OpenVPN username and password to that of your provider, and you'll also need to set a username and password for your transmission web console.

For now, you also want to keep `OPENVPN_CONFIG` as `dummy`. Every provider has their own set of values to put in here and by leaving it as dummy, you'll get a list of possible endpoints in your container logs upon first run. Once you get this, you can put a comma separated list of the endpoints you'd like in this field. Upon start, the container will pick a random one from the list.


Here's an example:
```env
#Global
TZ=America/Los_Angeles
PUID=1000
PGID=1000
USVC=1000
GSVC=1000

#Directories
APPDATA=/opt/dockerdata/  # All your docker data lives here
GMEDIA=/mnt/gdrive/:/gmedia  # Plex media root folder
DOWNLOAD=/opt/downloads/:/download  # Transmission download directory
WATCH=/opt/downloads/watch/:/watch  # Tranmission torrent watch directory
GMEDIA_TV=/mnt/gdrive/TV/:/tv  # Plex media folder for TV
GMEDIA_MOVIE=/mnt/gdrive/Movies/:/movies  # Plex media folder for Movies

# Torrents
OPENVPN_USER=MyUsername  # OpenVPN username (get from provider)
OPENVPN_PASS=password123  # OpenVPN password (get from provider)
OPENVPN_CONFIG=dummy  # OpenVPN endpoints (keep this as dummy for now)
OPENVPN_PROVIDER=WINDSCRIBE  # OpenVPN provider name
TRANSMISSION_USER=admin # Transmission web console username
TRANSMISSION_PASS=password123  # Transmission web console password

# Remote access
WIREGUARD_PEERS=2  # Number of peers wireguard will generate on first run
```


### Build and run containers
Once you've edited the `.env` file to your liking, just run the following to build and run the stack:
```
cd ~/plex-setup/docker/
docker-compose -f compose.yml up -d
```

Since we've included `Portainer` in the stack, you can navigate to port 9000 on your host and manage the containers from there.

## Post-setup Stuffs
At this point, if all went well, you're pretty much done and can stop reading here if you want!  
Yea. It was literally that easy!

For the sake of being detailed (because I'm sure I'll mess something up and come back to this) I'll explain my setup of each image and its use as well as a few extra things.

### Coming from a previous plex install
If you had a previous Plex install, you'll need to transfer the data over using [this guide](https://support.plex.tv/articles/201370363-move-an-install-to-another-system/).

Your new plex folder will be wherever you set `APPDATA` to in the `.env` file. There will be a plex folder inside there.

## Container Details
Going back to that diagram from eariler, here's a quick explanation of what all the parts are doing and why. Click the names for more in-depth explanations of what these programs are.

- [Jackett](https://github.com/Jackett/Jackett): Adds more trackers to Sonarr and Radarr
- [Sonarr](https://github.com/Sonarr/Sonarr): Grabs TV
- [Radarr](https://github.com/Radarr/Radarr): Grabs Movies
- [Transmission + OpenVPN](https://github.com/haugene/docker-transmission-openvpn): Transmission with OpenVPN and used to hide torrent traffic behind VPN provider
- [Plex](https://www.plex.tv/): Allows me to self host my media with a pretty UI
- [Tautulli](https://github.com/Tautulli/Tautulli): Cool usage stats for Plex
- [Overseerr](https://github.com/sct/overseerr): Web dashboard for users to request new content
- [Nginx Proxy Manager](https://github.com/jc21/nginx-proxy-manager): Allows me to forward overseerr to a website url
- [Wireguard](https://www.wireguard.com): VPN allowing me to access this home server's network when I'm away
- [Watchtower](https://github.com/containrrr/watchtower): Automate docker container base image updates
- [Portainer](https://github.com/portainer/portainer): Docker on ez-mode

So with that in mind, Jackett extends the search capabilities of Sonarr and Radarr. Those two then use the Transmission + OpenVPN container to acquire content and then it gets imported into Plex. Overseerr actually controls both of these services and provides and even higher level of abstraction. From the Overseerr dashboard, I or anyone I've given access to can request content, and as soon as it's approved, the signal is sent to either Sonarr or Radarr to fetch it. Even the automation got automated! Oh, and Nginx Proxy Manager is essentially Nginx with a pretty dashboard that makes things ez-pz and I use that to forward Overseerr to my domain so that it's accessible online. I think the rest is fairly self explanatory given the summaries above. Those pieces are mainly to make my life easier when managing the other images.

If you need any help configuring some of these programs, check [these guides](https://github.com/TRaSH-/Guides) out!

## Conclusion
There you have it! A way to easily automate plex via docker containers and also not alert your ISP to your shenanigans!
I've been using this for a while now and the peace of mind I get when my server is accidentally restarted or some other error occurs is amazing. If a piece in the system breaks, I just hop onto the portainer dashboard and I'm able to check logs, restart or recreate the container, and bunch more. Hopefully this helps someone acheive the same and makes their life easier!

Peace peace,  
Swap