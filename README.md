# Jellyfin-project
Media Server implementation to practice system administration and solution architect roles. 

## Purpose 
Purpose of this project is to provide a 'One For All' streaming service for my family and friends to access anywhere in the world! 

Secondly, the objective of this readme is to provide a step-by-step instructions to replicate the deployment of the infrastructure and used as a guide for troubleshooting.

## Technology Stack
**Server**
* OS - Ubuntu Desktop
* Remote protocol - SSH
* Virtualised solution - Docker
* Utilisation monitor tool - htop
* File shares - Mounted volumes from NAS

**Container applications**
* Jellyfin - Media streaming application
* Prowlarr - indexers management app
* Sonarr - tv-show query + downloads
* Radarr - movies query + downloads
* qBittorrent - access to ocean feature
* Homarr - GUI Dashboard for monitoring docker containers
* OpenVPN - To establish secured private conenction between containers and traffic from the internet.

## Current Roadblock
Container config issues with deployment for:
* nginx - mime.types syntax issue
* Certbot - [Errno 13] Permission denied: '/var/log/letsencrypt/.certbot.lock

**Current status of project**: Able to stream within local network only, until nginx and certbot issue is resolved. 


## Lets get started!
**NOTE** - Please change naming of files and dir for security reasons.

### Step 1 - Docker Setup
1.1 Update package list 
```
 sudo apt update
```
1.2. Install prerequisites
```
 sudo apt install -y ca-certificates curl gnupg
```

1.3. Adding Docker's official GPG key
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
1.4. Setup Docker repo
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
1.5. Install docker engine
```
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
1.6. Verify Installation
```
 docker --version
```
1.7. Install software packages on docker
```
docker pull jellyfin/jellyfin:latest
docker pull lscr.io/linuxserver/prowlarr:latest
docker pull lscr.io/linuxserver/sonarr:latest
docker pull lscr.io/linuxserver/radarr:latest
docker pull lscr.io/linuxserver/qbittorrent:latest
docker pull kylemanna/openvpn-client:latest
```
1.8. Confirm pulled images
```
docker images
```

### Step 2 - Mount Synology NAS to Server
2.1. Installing 'cifs-utils' for SMB support in files sharing with the NAS.
```
 sudo apt install -y cifs-utils

```
2.2. Creating the Mount Points
```
sudo mkdir -p /mount/NAS_Server/Movies
sudo mkdir -p /mount/NAS_Server/Shows
sudo mkdir -p /mount/NAS_Server/Download
```

2.3. Mount the NAS
**NOTE**
NAS_IP= your external storage device private IPv4
your_username & your_password = local user/admin credentials of the storage device

```
sudo mount -t cifs //NAS_IP/NAS_Server/Movies /mount/NAS_Server/Movies -o username=your_username,password=your_password

sudo mount -t cifs //NAS_IP/NAS_Server/Shows /mount/NAS_Server/Shows -o username=your_username,password=your_password

sudo mount -t cifs //NAS_IP/NAS_Server/Download /mount/NAS_Server/Download -o username=your_username,password=your_password
```
2.4 Verify the Mounts are Present
```
df -h
```

2.5 Persist Mounts across Reboots
edit the 'fstab.bak' file to automatically mount the fodlers after a reboot of the server. 
```
 sudo cp /etc/fstab /etc/fstab.bak
 sudo nano /etc/fstab
```

Add the following within the fstab file (can add anywhere):
**NOTE**
your_username & your_password = relates to the user credential of the storage device
```
//NAS_IP/NAS_Server/Movies /mount/NAS_Server/Movies cifs username=your_username,password=your_password,iocharset=utf8 0 0
//NAS_IP/NAS_Server/Shows /mount/NAS_Server/Shows cifs username=your_username,password=your_password,iocharset=utf8 0 0
//NAS_IP/NAS_Server/Download /mount/NAS_Server/Download cifs username=your_username,password=your_password,iocharset=utf8 0 0
```

2.6. Verify fstab and Mounts

```
 sudo mount -a
```

check mounts
```
ls /mount/NAS_Server/Movies
ls /mount/NAS_Server/Shows
ls /mount/NAS_Server/Download
```

2.7. Permission Update on Mounts
```
sudo chmod -R 755 /mount/NAS_Server
sudo chown -R $USER:$USER /mount/NAS_Server
```

### Step 3 - Deploy Jellyfin Pipeline Containers
We will use a docker-compose.yml file to simplify the deployment and management of all containers. 
Using Compose ensures that all container settings, volumes, and dependencies are centrally managed.

3.1. Create container directory and open dir
```
sudo -p mkdir ~/container-pipeline/ && cd ~/container-pipeline/
```

3.2. Install docker-compose 
```
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
Set executable Permission
```
sudo chmod +x /usr/local/bin/docker-compose
```
Verify install
```
docker-compose --version
```
pull required images
```
sudo docker-compose pull
```

3.3. Create docker-compose file
Docker-compose file is used when a docker container is deployed as it houses all the container's configuration.  

```
sudo nano docker-compose.yml
```
Content of 'docker-compose':
```
version: "3.8"

services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    ports:
      - "8096:8096"  # LAN access
    volumes:
      - /mount/Movies:/media/movies:ro
      - /mount/NAS_Server/Shows:/media/shows:ro
    restart: unless-stopped
    environment:
      - TZ=Australia/Sydney

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    ports:
      - "9696:9696"
    volumes:
      - ~/jellyfin-pipeline/prowlarr:/config
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Australia/Sydney

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    ports:
      - "8989:8989"
    volumes:
      - ~/jellyfin-pipeline/sonarr:/config
      - /mount/NAS_Server/Shows:/shows
      - /mount/NAS_Server/Download:/downloads
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Australia/Sydney

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    ports:
      - "7878:7878"
    volumes:
      - ~/jellyfin-pipeline/radarr:/config
      - /mount/NAS_Server/Movies:/movies
      - /mount/NAS_Server/Download:/downloads
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Australia/Sydney

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    ports:
      - "8080:8080"
      - "6881:6881/udp"
    volumes:
      - ~/jellyfin-pipeline/qbittorrent:/config
      - /mount/NAS_Server/Download:/downloads
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Australia/Sydney
      - WEBUI_PORT=8080
      - VPN_ENABLED=true  # Enabled by default
      - VPN_PROVIDERS= your_provider #Example OpenVPN, ExpressVPN, PIA, etc.
      - VPN_USERNAME=your_vpn_username
      - VPN_PASSWORD=your_vpn_password

  openvpn:
    image: kylemanna/openvpn
    container_name: openvpn
    cap_add:
      - NET_ADMIN
    ports:
      - "1194:1194/udp"
    volumes:
      - ~/jellyfin-pipeline/openvpn:/etc/openvpn
    restart: unless-stopped
    environment:
      - TZ=Australia/Sydney
```

3.4. Deploying the containers
```
sudo docker-compose up -d 
```
**Some tips for troubleshooting:**

Check on status of containers
```
sudo docker ps
```
check logs of a container 
```
sudo docker logs container_name
```


3.5. Permission for containers to read, write, and execute
**Container Default User** is '1000:1000' 
**775** - 7-owner, 7-groups, 5-others, where **7**=can read, write, & exe while **5**=read and exe only
```
sudo chown -R 1000:1000 /mount/NAS_Server/Movies /mount/NAS_Server/Shows /mount/NAS_Server/Download

sudo chmod -R 775 /mount/NAS_Server/Movies /mount/NAS_Server/Shows /mount/NAS_Server/Download

```


### Step 4 - Setting up Jellyfin, Prowlarr, Sonarr, & Radarr

### Step 5 - Security
- fail2ban 
- nginx and certbot setup

### Adding (Optional)

### Step 6 - Remote Access (WIP)
- nginx 
- certbot
- duckdns