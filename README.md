# automated-plex-guide
A complete guide to setting up a Plex home media server with automated requests/downloads

# Prerequisites
This guide is written for Ubuntu 20.04.1 LTS. While I expect it to work on most distros since all of the services are just running in docker containers I have not tested it on other distros and cannot provide much support if you run into issues.

# The Stack
Here's the stack we'll be using. There will be a section describing the installation and configuration for each one of these :)

**Docker** lets us run and isolate each of our services into a container. Everything for each of these services will live in the container except the configuration files which will live on our host.

**Plex** is a "client-server media player system". There are a few alternatives, but I chose Plex here because they have a client available on nearly every platform.

**Qbittorrent** is a torrent client. This can be substituted for Deluge or Transmission but I personally prefer Qbittorrent.

**Jackett** is a tool that Sonarr and Radarr use to search indexers and trackers for torrents

**Sonarr** is a tool for automating and managing your TV library. It automates the process of searching for torrents, downloading them then "moving" them to your library. It also checks RSS feeds to automatically download new shows as soon as they're uploaded! 

**Radarr** Is a fork of Sonarr that does all the same stuff but for Movies

**Ombi** is a super simple web UI for sending requests to Radarr and Sonarr

# Tips for managing your hard drives (Optional)

## Optimizing for Ratios

If you want to use private trackers to get your torrents you will need to maintain a high seeding ratio. If you aren't familiar with this term, it basically means you should be uploading as much, if not more than you download.

The best way I've found to do this is to mount your drives directly to the machine that handles your downloads. This means you can configure Sonarr and Radarr to create hardlinks when a torrent finishes. With this enabled, a reference to the data will exist in your media directory and in your torrent directory so Transmission can continue seeding everything you download.

## HDD vs SSD

Can also be written as Space vs. Reliability and Speed. I'm a freaking hoarder when it comes to media now, so I go with HDDs. This means I need to worry about my drives randomly dying.

## HDD Backups vs. Redundancy

HDDs are very prone to randomly dying and you need to prepare for this. One way you can prepare for this is to setup a RAID array. (Not covered in this guide)

# Installing Docker

This is an easy one :)

```
sudo apt-get update
sudo apt install docker.io
sudo systemctl start docker
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o
/usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

I also like to keep my configs in one easy place in case I want to transfer them anywhere, so let’s create a folder to hold our docker container configs and create our docker-compose.yml file

```
mkdir ~/docker-services
touch ~/docker-services/docker-compose.yml
```

And add this to your docker-compose file. We will be filling in the services in the coming steps. If you are confused on how to add the services or how your file should look, [here is a good resource on docker-compose](https://docs.docker.com/compose/compose-file/compose-file-v2/).

```
---
version: "2.1"
services:

```

# Plex Docker Config

Add the following lines to ~/docker-services/docker-compose.yml under "services:"

```
  plex:
    image: ghcr.io/linuxserver/plex
    container_name: plex
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - VERSION=docker
      - PLEX_CLAIM= #optional
    volumes:
      - ~/docker-services/config/plex:/config
      - /path/to/media:/media
    restart: unless-stopped
```

This will start your Plex server on port 32400, and add the volumes ~/docker-services/plex/config and /path/to/media onto the container. Replace /path/to/media with your prefered location. If you are trying to move your current Plex configs over, run something like this

```
mv /var/lib/plexmediaserver/* ~/docker-services/plex/config/
```

Note that plex is looking for your config directory to contain a single directory Library. Look for that directory and copy it over.

If you are on something other than Ubuntu, [refer to this page to find your configs.](https://support.plex.tv/articles/202915258-where-is-the-plex-media-server-data-directory-located/)

# Qbittorrent Docker Config

Add the following lines to ~/docker-services/docker-compose.yml under "services:"

```
qbittorrent:
    image: ghcr.io/linuxserver/qbittorrent
    container_name: qbittorrent
    network_mode: "host"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America\New_York
      - UMASK_SET=022
      - WEBUI_PORT=8080
    volumes:
      - ~/docker-services/config/qbittorrent:/config
      - /path/to/downloadclient-downloads:/downloads
    ports:
      - 6881:6881
      - 6881:6881/udp
      - 8080:8080
    restart: unless-stopped
 ```
 
Notice how we mount our torrent drive on the container in the same location as the host, rather than something like /downloads (which is suggested over at linuxserver). This, plus the config below ensures Sonarr and Radarr send torrents to the right directory.

Once Qbittorrent has started for the first time, it will make a settings.json file for you. Here are some changes you should make to the settings file. Come back here later to make these changes once everything is started, then run docker restart qbittorrent.

# Jackett Docker Config

Add the following lines to ~/docker-services/docker-compose.yml under "services:"

```
  jackett:
    image: ghcr.io/linuxserver/jackett
    container_name: jackett
    network_mode: "host"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=EST
      - AUTO_UPDATE=true #optional
      - RUN_OPTS=<run options here> #optional
    volumes:
      - ~/docker-services/config/jackett:/config
      - /path/to/downloadclient-downloads:/downloads
    ports:
      - 9117:9117
    restart: unless-stopped
```

This is super basic and just boots your Jackett service on port 9117. Doesn’t need much else!

# Sonarr & Radarr Docker Config

Add the following lines to ~/docker-services/docker-compose.yml under "services:"

```
  sonarr:
    image: ghcr.io/linuxserver/sonarr
    container_name: sonarr
    network_mode: "host"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=EST
    volumes:
      - ~/docker-services/config/sonarr:/config
      - /path/to/tvseries:/tv
      - /path/to/downloadclient-downloads:/downloads
    ports:
      - 8989:8989
    restart: unless-stopped
  radarr:
    image: ghcr.io/linuxserver/radarr
    container_name: radarr
    network_mode: "host"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=EST
    volumes:
      - ~/docker-services/config/radarr:/config
      - /path/to/movies:/movies
      - /path/to/downloadclient-downloads:/downloads
    ports:
      - 7878:7878
    restart: unless-stopped
 ```
 
Now, these guys are freaking TRICKY. Make sure those PUID and GUID match the ID for your user and group… and make sure that user has read-write permissions for /mnt/media. Sonarr and Radarr are going to try to be creating folders and files in there when they copy or hard link files over.

If you are running into issues, check the logs of the docker container or the logs in the web UI. It should tell you exactly where it’s having trouble. Then log into the user you set it to run as an attempt the same actions. See whats going on first hand.

# Ombi Docker Config

Add the following lines to ~/docker-services/docker-compose.yml under "services:"

```
  ombi:
    image: ghcr.io/linuxserver/ombi
    container_name: ombi
    network_mode: "host"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=EST
      - BASE_URL=/ombi #optional
    volumes:
      - ~/docker-services/config/ombi:/config
    ports:
      - 3579:3579
    restart: unless-stopped
 ```
 
This will open Ombi on port 3579 but sadly they don’t have SSL support by default. I can create a post on adding LetsEncrypt to this setup if it get’s enough traction!

# Start it up!

Run this command to boot up all your services! Remember to go back and update your Transmission settings after this.

```
cd ~/docker-services
docker-compose up -d
```

You now have these services running locally. Go to their web UI’s and configure everything.

```
sonarr       @ http://localhost:8989
radarr       @ http://localhost:7878
jackett      @ http://localhost:9117
qbittorrent  @ http://localhost:8080
plex         @ http://localhost:32400
ombi         @ http://localhost:3579
```

# Configuration
## qBittorrent
### Bind qBittorrent to VPN

```
1. Navigate to qBittorrent Web UI at http://localhost:8080
2. Click on Tool > Options
3. Go to the Advanced tab
4. Change Network interface from Any to tun0 (or whichever interface corresponds to your VPN)
```
## Jackett
### Configure Jackett with your indexers

Login to the Jackett Web UI at http://localhost:9117. Once there click on "Add Indexer" and find your favorite torrenting sites. Personally I like to use 1337x, The Pirate Bay, and RARGB. Once you find the indexor you would like to setup click on the green plus button on the right. If you choose a private tracker you will first have to login with your credentials.

## Sonarr / Radarr
### Adding a Jackett indexer

Repeat for each indexor

```
1. Navigate to Settings > Indexers > Add > Torznab > Custom
2. Enter an appropriate name for the indexor in the "Name" field
3. Keep checkboxes set to default values
4. Find the ocrresponding indexor in Jackett and click "Copy Torznab Feed" then paste into Sonnar/Radarr URL field.
5. Refer to Jackett UI for API Key
6. Select the categories that you would like to use this indexor for (I just select "TV" for any indexors in Sonarr and "Movies" for any indexors in Radarr)
```

### Connect to qBittorrent

```
1. Navigate to Settings > Downloads Clients > Add
2. Select "qBittorrent"
3. Enter "qBittorrent" in the "Name" field
4. Set host and port to the qBittorrent Web UI. If you followed this guide correctly the default values should be correct.
5. (Optional) Select SSL and follow the instructions next to the checkbox
6. Enter the Username and Password for the webui. By default the username is "admin" and the password is "adminadmin"
7. Click "Save"
```

### Set root directory

```
1. Navigate to Settings > Media Management
2. Scroll to the bottom of the page and click on "Add Root Folder"
3. Sonarr: Select /tv
4. Radarr: Select /movies
```

### (Optional) File renaming

```
1. Navigate to Settings > Media Management
2. Check the checkbox under "Rename Episodes/Movies"
3. (Optional) Configure episode format by preference
```
## Ombi

### Initial Setup

When you first navigate to the Ombi Web UI it will walk you through a setup wizard.

For Media Server select Plex and login with your Plex credentials.

You will also need to create a local admin account.

(Optional) Customize to your preference.

### Connect Sonarr and Radarr

**Sonarr:**
```
1. Navigate to Settings > TV > Sonarr
2. Check "Enable" "V3" and "Scan for Availability"
3. Enter hostname and port. If you followed this guide correctly it should just be localhost:8989
4. Enter API key. You can find this in Sonarr under Settings > General
5. Leave base URL blank unless you configured one in Sonarr
6. Under Sonarr Interface click on "Load Qualities", "Load Folders", and "Load Languages"
7. Select your prefered quality under Quality Profiles
8. Select /tv under Default Root Folders
9. Select your prefered languages under Language Profiles
```

**Radarr:**

```
1. Navigate to Settings > Movies > Radarr
2. Check "Enable" "V3" and "Scan for Availability"
3. Enter hostname and port. If you followed this guide correctly it should just be localhost:7878
4. Enter API key. You can find this in Radarr under Settings > General
5. Leave base URL blank unless you configured one in Radarr
6. Under Radarr Interface click on "Load Profiles" and "Load Root Folders"
7. Select your prefered quality under Quality Profiles
8. Select /movies under Default Root Folders
9. Select Physical / Web under Default Minimum Availability. Optionally you could select and earlier setting in case a movie gets leaked before being released to DVD but you will more often than not probably just get cam recordings.
```

 
