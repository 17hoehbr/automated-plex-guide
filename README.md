# automated-media-server-guide (WIP)

# In the process of updating

A complete guide to setting up a home media server with automated requests/downloads

Based on the [blog post](https://zacholland.net/a-complete-guide-to-setting-up-a-plex-home-media-server-with-automated-requests-downloads/) from Zac Holland. Most of the information is the same but I've added more details on the configuration side and will be adding an updated section on managing multiple HDDs.

# Prerequisites

This guide has been tested on Ubuntu Server 21.10. Most of it should work just fine on any distro, but if you're using a non-debian distro you'll have to look up distro-specific instructions for installing docker. Once installed though the rest of the guide should still work.

# The Stack

Here's the stack we'll be using. There will be a section describing the installation and configuration for each one of these :)

**Docker** lets us run and isolate each of our services into a container. Everything for each of these services will live in the container except the configuration files which will live on our host.

**Jellyfin** is a completely open sourced alternative to Plex. The interface is not quite as polished and client support is slightly limited, however all of it's features are completely free and users can contribute to or modify the code as they see fit. This can allow for implementation of features or changes to the server that could not be achieved with Plex.

**qBittorrent** is a torrent client. Transmission and Deluge are also popular choices but I chose qBittorrent because you can easily configure it to only operate over the VPN connection.

**openvpn-client** is a VPN running in docker. This will allow you to connect the qBittorrent container to your VPN without having to put your entire system behind it

**Prowlarr** is a tool that Sonarr and Radarr use to search indexers and trackers for torrents

**Sonarr** is a tool for automating and managing your TV library. It automates the process of searching for torrents, downloading them then "moving" them to your library. It also checks RSS feeds to automatically download new shows as soon as they're uploaded! 

**Radarr** is a fork of Sonarr that does all the same stuff but for Movies

## Optional

**[jfa-go](https://github.com/hrfee/jfa-go)** is a user manager for Jellyfin that allows your users to sign up via an invite code and reset their passwords

**[Organizr](https://github.com/causefx/Organizr)** is a dashboard for keeping track all of these web services

**[Bazarr](https://wiki.bazarr.media/Getting-Started/Setup-Guide/)** is a tool for Sonarr and Radarr to download subtitles for your content

**[nginx-proxy-manager](https://nginxproxymanager.com/guide/#quick-setup)** is a simple reverse proxy service for making Jellyfin accessible outside of your local network

# Installing Docker


```
sudo apt update
sudo apt install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
```

To verify that the docker installation was successful, run the following command:
```
docker container run hello-world
```

The output should look something like [this](https://linuxize.com/post/how-to-install-and-use-docker-on-ubuntu-20-04/docker-hello-world_hu0e93396502d67b4a7c43a5ce3cac5c2c_75442_768x0_resize_q75_lanczos.jpg)

I also like to keep my configs in one easy place, so let’s create a folder to hold our docker container configs and create our docker-compose.yml file

```
mkdir ~/docker
touch ~/docker/docker-compose.yml
```

And add this to your docker-compose file. We will be filling in the services in the coming steps. If you are confused on how to add the services or how your file should look, [here is a good resource on docker-compose](https://docs.docker.com/compose/compose-file/compose-file-v2/).

```
---
version: "2.1"
services:

```

# Jellyfin Docker Config

Add the following lines to ~/docker/docker-compose.yml under "services:"

```
  jellyfin:
    image: ghcr.io/linuxserver/jellyfin
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - ~/docker/config/jellyfin:/config
      - /path/to/media:/media
    ports:
      - 8096:8096
    restart: unless-stopped
```

This will start your Jellyfin server on port 8096, and add the volumes ~/docker/config/jellyfin and /path/to/media onto the container.

Replace /path/to/media to your media directory. Your media directory should contain subdirectories labelled "tv" and "movies".

Make sure those PUID and GUID match the ID for your user and group. You can find these by simply typing "id" into terminal. If they don't match simply replace the PUID and GUID in the docker compose script with the ones you found in terminal.

# VPN Docker Config

These instructions will vary depending on your VPN provider.




# qBittorrent Docker Config

# Note to self: update with VPN configuration

```
qbittorrent:
    image: ghcr.io/linuxserver/qbittorrent
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - WEBUI_PORT=8080
    volumes:
      - ~/docker/config/qbittorrent:/config
      - /path/to/qbittorrent-downloads:/downloads
    network_mode: container:mullvad-client
    restart: unless-stopped
 ```

Replace /path/to/qbittorrent-downloads with the directory you would like qBittorrent to store downloads.


# Prowlarr Docker Config

```
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:develop
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - ~/docker/config/prowlarr:/config
    ports:
      - 9696:9696
    restart: unless-stopped
```

This is super basic and just boots your Prowlarr service on port 9696. Doesn’t need much else!

Make sure those PUID and GUID match the ID for your user and group. You can find these by simply typing "id" into terminal. If they don't match simply replace the PUID and GUID in the docker compose script with the ones you found in terminal.


# Sonarr & Radarr Docker Config

```
  sonarr:
    image: ghcr.io/linuxserver/sonarr
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=EST
    volumes:
      - ~/docker/config/sonarr:/config
      - /path/to/media/tv:/tv
      - /path/to/qbittorrent-downloads:/downloads
    ports:
      - 8989:8989
    restart: unless-stopped
  radarr:
    image: ghcr.io/linuxserver/radarr
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=EST
    volumes:
      - ~/docker/config/radarr:/config
      - /path/to/media/movies:/movies
      - /path/to/qbittorrent-downloads:/downloads
    ports:
      - 7878:7878
    restart: unless-stopped
 ```
Replace /path/to/media/tv & /path/to/media/movies with the directories you created during the Plex docker setup. Replace /path/to/qbittorrent-downloads with the directory you created during the qBittorrent docker setup.

Make sure those PUID and GUID match the ID for your user and group. You can find these by simply typing "id" into terminal. If they don't match simply replace the PUID and GUID in the docker compose script with the ones you found in terminal.

Also make sure that user has read-write permissions for the media directories. Sonarr and Radarr are going to try to be creating folders and files in there when they copy or hard link files over. You can do this with the following command: ```chmod -r +rw /path/to/media```

If you are running into issues, check the logs of the docker container or the logs in the web UI. It should tell you exactly where it’s having trouble. Then log into the user you set it to run as an attempt the same actions. See whats going on first hand.

# Start it up!

Run this command to boot up all your services! Remember to go back and update your qBittorrent settings after this.

```
cd ~/docker
docker-compose up -d
```

You now have these services running locally. Go to their web UI’s and configure everything.

```
sonarr       @ http://localhost:8989
radarr       @ http://localhost:7878
prowlarr     @ http://localhost:9696
qbittorrent  @ http://localhost:8080
jellyfin     @ http://localhost:8096
```

# Configuration

## Jellyfin

1. Navigate to http://localhost:8096
2. Follow the setup wizard
3. Add libraries for TV and Movies. The library folders should be located at /media/movies and /media/tv respectively.

See [documentation](https://jellyfin.org/docs/general/quick-start.html) for more information
## qBittorrent

### Set Download Directory

1. Navigate to qBittorrent Web UI at http://localhost:8080
2. Click on Tools > Options
3. Under the "Downloads" tab change "Default Save Path" to /downloads/completed
4. Check the box next to "Keep incomplete torrents in:" Make sure it's set to /downloads/incomplete

### Bind qBittorrent to VPN

This will ensure that qBittorrent only downloads over your VPN connection.

1. Navigate to qBittorrent Web UI at http://localhost:8080
2. Click on Tool > Options
3. Go to the Advanced tab
4. Change Network interface from Any to the interface corresponding to your VPN (probably something like tun0)

### Change default login (Optional)

By default the login to the web UI is just "admin" "adminadmin". Personally I don't find it worth changing since qBittorrent is only accessible through my home network but if you plan on making it available remotely you might want to change it.

1. Navigate to Tools > Options
2. Under the "Downloads" tab scroll down to "Authentication"
3. Enter a secure username and password.

## Prowlarr

See [Prowlarr Quick Start Guide](https://wiki.servarr.com/prowlarr/quick-start-guide)

## Sonarr / Radarr

### Connect to qBittorrent

1. Navigate to Settings > Downloads Clients > Add
2. Select "qBittorrent"
3. Enter "qBittorrent" in the "Name" field
4. Set host and port to the qBittorrent Web UI. If you followed this guide correctly the default values should be correct
5. Enter the Username and Password for the webui
6. Click "Save"


### Set root directory

1. Navigate to Settings > Media Management
2. Scroll to the bottom of the page and click on "Add Root Folder"
3. Sonarr: Select /tv
4. Radarr: Select /movies

### Create quality profile

You will want to create a quality profile to specify what resolution you want your TV Shows and Movies to be.

1. Navigate to Settings > Profiles
3. Select the Any profile
4. Check all qualities you would like to allow and uncheck all qualities you would like to disable. For example if you want your movie quality to cap out at 1080p to save disk space uncheck everything above Bluray-1080p. Sonarr/Radarr will prioritize the highest allowed resolution but will download lower allowed ones if it can't find it.
5. (Optional) Allow upgrades by checking the "Upgrades Allowed" Checkbox. You can then change the "Upgrade Until" drop down to your prefered maximum resolution. This is useful for if you want your library to be entirely 1080p but the only torrent available for a specific show or movie is 720p. This way it will still download the 720p torrent but if a 1080p torrent ever comes along it will automatically download it and replace the 720p version.


### File renaming (Optional)

I like having this setting enabled to keep my media folders nice and organized.

1. Navigate to Settings > Media Management
2. Check the checkbox under "Rename Episodes/Movies"
3. (Optional) Configure episode format by preference

## Ombi

### Initial Setup

When you first navigate to the Ombi Web UI it will walk you through a setup wizard.

For Media Server select Plex and login with your Plex credentials.

You will also need to create a local admin account.

(Optional) Customize to your preference.

### Connect Sonarr and Radarr

**Sonarr:**

1. Navigate to Settings > TV > Sonarr
2. Check "Enable" "V3" and "Scan for Availability"
3. Enter hostname and port. If you followed this guide correctly it should just be localhost:8989
4. Enter API key. You can find this in Sonarr under Settings > General
5. Leave base URL blank unless you configured one in Sonarr
6. Under Sonarr Interface click on "Load Qualities", "Load Folders", and "Load Languages"
7. Select "Any" under Quality Profiles
8. Select /tv under Default Root Folders
9. Select your prefered languages under Language Profiles


**Radarr:**


1. Navigate to Settings > Movies > Radarr
2. Check "Enable" "V3" and "Scan for Availability"
3. Enter hostname and port. If you followed this guide correctly it should just be localhost:7878
4. Enter API key. You can find this in Radarr under Settings > General
5. Leave base URL blank unless you configured one in Radarr
6. Under Radarr Interface click on "Load Profiles" and "Load Root Folders"
7. Select "Any" under Quality Profiles
8. Select /movies under Default Root Folders
9. Select Physical / Web under Default Minimum Availability. Optionally you could select and earlier setting in case a movie gets leaked before being released to DVD but you will more often than not probably just get cam recordings.


### Connect Jellyfin

Settings > Media Server > Jellyfin

Click Add Server

Then make sure to click "Load Libraries" and check all relevant ones.

(Optional)

### OMBI 

Finally you will want to create Ombi accounts for your users so they can submit requests. You can do this manually under the "User Management" tab, but that requires you to manually create an account for everyone connected to your Plex.

Instead you can allow users to authenticate with their Plex login by going to Settings > Configuration > Authentication and selecting "Enable Plex OAuth". This way anyone who has access to your Plex can simply login with their Plex credentials and make requests.


Once the setup is complete you can login with either the local account you made earlier or your Plex account.
