# automated-media-server-guide (WIP)

# Very outdated, might update eventually

A complete guide to setting up a home media server with automated requests/downloads

Based on the [blog post](https://zacholland.net/a-complete-guide-to-setting-up-a-plex-home-media-server-with-automated-requests-downloads/) from Zac Holland. Most of the information is the same but I've added more details on the configuration side and will be adding an updated section on managing multiple HDDs.

# Prerequisites
This guide is written for Ubuntu Server 20.04.1 LTS. While I expect it to work on most distros since all of the services are just running in docker containers I have not tested it on other distros and cannot provide much support if you run into issues.

# The Stack
Here's the stack we'll be using. There will be a section describing the installation and configuration for each one of these :)

**Docker** lets us run and isolate each of our services into a container. Everything for each of these services will live in the container except the configuration files which will live on our host.

**Plex** is a "client-server media player system". Plex is overall the most mature media server at the moment but some features are locked behind the paid Plex Pass.

**Jellyfin** is a completely open sourced alternative to Plex. The interface is not quite as polished and client support is slightly limited, however all of it's features are completely free and users can contribute to or modify the code as they see fit. This can allow for implementation of features or changes to the server that could not be achieved with Plex.

**qBittorrent** is a torrent client. While you could use Transmission or Deluge instead I opted for qBittorrent because you can set it to only download from the VPN interface so you don't accidently expose yourself to your ISP.

**Jackett** is a tool that Sonarr and Radarr use to search indexers and trackers for torrents

**Sonarr** is a tool for automating and managing your TV library. It automates the process of searching for torrents, downloading them then "moving" them to your library. It also checks RSS feeds to automatically download new shows as soon as they're uploaded! 

**Radarr** is a fork of Sonarr that does all the same stuff but for Movies

**Ombi** (Optional) is a super simple web UI for allowing your users to send requests to Radarr and Sonarr

**Tautulli** (Optional) provides some neat analystics from Plex. Useful for analysing which content your users use the most.


# Installing Docker

This is an easy one :)

```
sudo apt update
sudo apt install docker.io
sudo systemctl start docker
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

I also like to keep my configs in one easy place in case I want to transfer them anywhere, so let’s create a folder to hold our docker container configs and create our docker-compose.yml file

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

# Plex Docker Config

**Use only Plex OR Jellyfin**

Add the following lines to ~/docker/docker-compose.yml under "services:"

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
      - ~/docker/config/plex:/config
      - /path/to/media:/media
    restart: unless-stopped
```

This will start your Plex server on port 32400, and add the volumes ~/docker/config/plex and /path/to/media onto the container.

Replace /path/to/media to your media directory. Your media directory should contain subdirectories labelled "tv" and "movies".


If you are trying to move your current Plex configs over, run something like this

```
mv /var/lib/plexmediaserver/* ~/docker/config/plex/
```

Note that plex is looking for your config directory to contain a single directory Library. Look for that directory and copy it over.

If you are on something other than Ubuntu, [refer to this page to find your configs.](https://support.plex.tv/articles/202915258-where-is-the-plex-media-server-data-directory-located/)

# Jellyfin Docker Config

**Use only Plex OR Jellyfin**

Add the following lines to ~/docker/docker-compose.yml under "services:"

```
  jellyfin:
    image: ghcr.io/linuxserver/jellyfin
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=EST
    volumes:
      - ~/docker/config/jellyfin:/config
      - /path/to/media:/media
    ports:
      - 8096:8096
    restart: unless-stopped
```

This will start your Jellyfin server on port 8096, and add the volumes ~/docker/config/jellyfin and /path/to/media onto the container.

Replace /path/to/media to your media directory. Your media directory should contain subdirectories labelled "tv" and "movies".

# qBittorrent Docker Config

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
      - ~/docker/config/qbittorrent:/config
      - /path/to/qbittorrent-downloads:/downloads
    ports:
      - 6881:6881
      - 6881:6881/udp
      - 8080:8080
    restart: unless-stopped
 ```

Replace /path/to/qbittorrent-downloads with the directory you would like qBittorrent to store downloads.

# Jackett Docker Config

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
      - ~/docker/config/jackett:/config
    ports:
      - 9117:9117
    restart: unless-stopped
```

This is super basic and just boots your Jackett service on port 9117. Doesn’t need much else!

# Sonarr & Radarr Docker Config

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
      - ~/docker/config/sonarr:/config
      - /path/to/media/tv:/tv
      - /path/to/qbittorrent-downloads:/downloads
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

# Ombi Docker Config

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
      - ~/docker/config/ombi:/config
    ports:
      - 3579:3579
    restart: unless-stopped
 ```
 
This will open Ombi on port 3579 but sadly they don’t have SSL support by default. I can create a post on adding LetsEncrypt to this setup if it get’s enough traction!

# Tautulli Docker Config

```
  tautulli:
    image: ghcr.io/linuxserver/tautulli
    container_name: tautulli
    network_mode: "host"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=EST
    volumes:
      - ~/docker/config/tautulli:/config
    ports:
      - 8181:8181
    restart: unless-stopped
```

Super simple, no configuration required.

# Start it up!

Run this command to boot up all your services! Remember to go back and update your Transmission settings after this.

```
cd ~/docker
docker-compose up -d
```

You now have these services running locally. Go to their web UI’s and configure everything.

```
sonarr       @ http://localhost:8989
radarr       @ http://localhost:7878
jackett      @ http://localhost:9117
qbittorrent  @ http://localhost:8080
plex         @ http://localhost:32400 / jellyfin    @ http://localhost:8096
ombi         @ http://localhost:3579
tautulli     @ http://localhost:8181
```

# Configuration

## Plex

1. Navigate to http://localhost:32400/web
2. Follow the setup wizard
3. Add libraries for TV and Movies. The library folders should be located at /media/movies and /media/tv.

## Jellyfin

1. Navigate to http://localhost:8096
2. Follow the setup wizard
3. Add libraries for TV and Movies. The library folders should be located at /media/movies and /media/tv.


## qBittorrent

### Set Download Directory

1. Navigate to qBittorrent Web UI at http://localhost:8080
2. Click on Tools > Options
3. Under the "Downloads" tab change "Default Save Path" to /downloads/completed
4. Check the box next to "Keep incomplete torrents in:" Make sure it's set to /downloads/incomplete

### Change default login (Optional)

By default the login to the web UI is just "admin" "adminadmin". Personally I don't find it worth changing since qBittorrent is only accessible through my home network but if you plan on making it available remotely you might want to change it.

1. Navigate to Tools > Options
2. Under the "Downloads" tab scroll down to "Authentication"
3. Enter a secure username and password.

### Bind qBittorrent to VPN

By binding qBittorrent to the VPN network interface you don't have to worry about torrents accidently slipping through to your ISP. With this enabled your downloads will automatically stop if you disconenct from your VPN instead of defaulting to your ISP's network.

This of course requires an active VPN subscription and for your server to be connected to it (Not covered in this guide)

1. Navigate to qBittorrent Web UI at http://localhost:8080
2. Click on Tool > Options
3. Go to the Advanced tab
4. Change Network interface from Any to the interface corresponding to your VPN (most likely tun0)

## Jackett
### Configure Jackett with your indexers

Login to the Jackett Web UI at http://localhost:9117.

Once there click on "Add Indexer" and find your favorite torrenting sites. Personally I like to use Kick Ass Torrents, 1337x, The Pirate Bay, and RARGB.

Once you find the indexor you would like to setup click on the green plus button on the right. If you choose a private tracker you will first have to login with your credentials.

## Sonarr / Radarr
### Adding a Jackett indexer

Repeat for each indexor

1. Navigate to Settings > Indexers > Add > Torznab > Custom
2. Enter an appropriate name for the indexor in the "Name" field
3. Keep checkboxes set to default values
4. Find the ocrresponding indexor in Jackett and click "Copy Torznab Feed" then paste into Sonnar/Radarr URL field.
5. Refer to Jackett UI for API Key
6. Select the categories that you would like to use this indexor for (I just select "TV" for any indexors in Sonarr and "Movies" for any indexors in Radarr)


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

### Connect Plex

Settings > Media Server > Plex

Fill out your Plex credentials on the right hand side and it should automatically fill the rest

Then make sure to click "Load Libraries" and check all relevant ones.

### Connect Jellyfin

Settings > Media Server > Jellyfin

Click Add Server

Then make sure to click "Load Libraries" and check all relevant ones.

### Create user profiles

Finally you will want to create Ombi accounts for your users so they can submit requests. You can do this manually under the "User Management" tab, but that requires you to manually create an account for everyone connected to your Plex.

Instead you can allow users to authenticate with their Plex login by going to Settings > Configuration > Authentication and selecting "Enable Plex OAuth". This way anyone who has access to your Plex can simply login with their Plex credentials and make requests.

## Tautulli (Optional)

Navigate to http://localhost:8181 and follow the setup wizard.

1. Create a local admin account
2. Sign in with Plex
3. Select your Plex server from the drop down. The rest of the settings should autofill. You can optionally select use secure connection but I'm not really sure what difference it makes since they're both on the same network anyway. Press verify and next.
4. The rest of the settings can be left as default

Once the setup is complete you can login with either the local account you made earlier or your Plex account.
