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
<details>
<summary>Step 1 - Setting up Docker</summary>

**1.1 Update package list**
```
 sudo apt update
```

**1.2. Install prerequisites**
```
 sudo apt install -y ca-certificates curl gnupg
```

**1.3. Adding Docker's official GPG key**
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

**1.4. Setup Docker repo**
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**1.5. Install docker engine**
```
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**1.6. Verify Installation**
```
 docker --version
```

**1.7. Install software packages on docker**
```
docker pull jellyfin/jellyfin:latest
docker pull lscr.io/linuxserver/prowlarr:latest
docker pull lscr.io/linuxserver/sonarr:latest
docker pull lscr.io/linuxserver/radarr:latest
docker pull lscr.io/linuxserver/qbittorrent:latest
docker pull kylemanna/openvpn-client:latest
```

**1.8. Confirm pulled images**
```
docker images
```
</details>

<details>

<summary>Step 2 - Mount Synology NAS to Server</summary>

**2.1. Installing 'cifs-utils' for SMB support in files sharing with the NAS.**
```
 sudo apt install -y cifs-utils

```

**2.2. Creating the Mount Points**
```
sudo mkdir -p /mount/NAS_Server/Movies
sudo mkdir -p /mount/NAS_Server/Shows
sudo mkdir -p /mount/NAS_Server/Download
```

**2.3. Mount the NAS**
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

**2.5 Persist Mounts across Reboots**
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

**2.7. Permission Update on Mounts**
```
sudo chmod -R 755 /mount/NAS_Server
sudo chown -R $USER:$USER /mount/NAS_Server
```
</details>

<details>

<summary> Step 3 - Deploy Jellyfin Pipeline Containers </summary>
We will use a docker-compose.yml file to simplify the deployment and management of all containers. 
Using Compose ensures that all container settings, volumes, and dependencies are centrally managed.

**3.1. Create container directory and open dir**
```
sudo -p mkdir ~/container-pipeline/ && cd ~/container-pipeline/
```

**3.2. Install docker-compose**
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

**3.3. Create docker-compose file**
Docker-compose file is used when a docker container is deployed as it houses all the container's configuration.  
```
sudo nano docker-compose.yml
```

**Content of docker-compose**: Copy and paste the content and adjust the volume pathing to your setup.
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

**3.4. Deploying the containers**
```
sudo docker-compose up -d 
```

**Some tips for troubleshooting:**
Check on status of containers
```
sudo docker ps
```

Check logs of a container
```
sudo docker logs container_name
```

**3.5. Permission for containers to read, write, and execute**
**Container Default User** is '1000:1000' 
**775** - 7-owner, 7-groups, 5-others, where **7**=can read, write, & exe while **5**=read and exe only
```
sudo chown -R 1000:1000 /mount/NAS_Server/Movies /mount/NAS_Server/Shows /mount/NAS_Server/Download

sudo chmod -R 775 /mount/NAS_Server/Movies /mount/NAS_Server/Shows /mount/NAS_Server/Download

```
</details>

<details>
  <summary>Step 4 - Setting up Jellyfin, Prowlarr, Sonarr, & Radarr</summary>

**4.1 Set up qBittorrent** 
qBittorrent container running for the first time will generate a random password for the container. - To see both the username and password details for qbittorrent follow the steps below.
```
docker logs <container_name> #example docker logs qbittorrent
```

**4.1.2 Qbittorrent login and change password**
```
http://<server_IP>:8080
```
Once logged in, navigate to option -> WebUI -> Authentication tab - include username and new password details. Hit Save located at the bottom of the prompt!

**4.2 Setting up Jellyfin app**
```
http://<server_IP>:8096
```
Follow the prompts and add in your mounts which will appear as /movies and /shows in the current displayed directory.

Test Jellyfin on your Smart TV by installing the Jellyfin app and pointing it to the local server IP.

**4.3 Setting up Prowlarr**
```
http://<server_IP>:9696
```
**4.3.1 Create your username and password**

**4.3.1 Linking with Radarr, Sonarr**
Settings -> Select Apps -> Add (Radarr and Sonarr) -> fill the following fields:
* Authentication : Forms (login page)
* Server address = url with the port eg. 192.168.1.2:8989 (Radarr).
* API keys= API keys can be found in settings/General in the respective apps.

**4.3.2 Linking to qbittorrent**
Settings/Download Clients -> Select qbittorrent -> include the following:
* Host = linux server
* Port = default is 8080 
* Username = qbittorrent user 
* Password = qbittorrent password

**4.4 Setting up Radarr and Sonarr**
Both services' setup will be identical so I will only focus on Sonarr here.
```
http://<server_IP>:7878 # Radarr
http://<server_IP>:8989 # Sonarr
```

**4.4.1 Adding Root folder and setup login page**
Within Settings/Media Management -> Root folders (add Root Folder) → in the main directory opened, select /movies.
General settings - within the Authentication field select Forms (login page).

**4.4.2 Adding Download Clients (qBittorrent)**
Under Download Clients select the add icon -> select qBittorrent -> fill the following fields:
* Host = linux server
* Port = default is 8080 
* Username = qbittorrent user 
* Password = qbittorrent password
* Hit the test button and then Save. 
**Note**: Any error displayed will indicate which field(s) you have incorrectly filled out and will be useful for troubleshooting. 

**Troubleshooting**
If there are permission issues with accessing root folders, try the following:
**1. Does the container user have same permission as host user?**
* open and edit your Docker-compose.yml file.
* Within the service sector **add user : "1000:1000"** to set container user to host admin and match same permissions to read, write or execute.
**NOTE** - defualt user ID is 1000 (most admins).
*Example*
```
homarr: 
container_name: homarr 
image: ghcr.io/ajnart/homarr:latest 
restart: unless-stopped 
volumes: 
- /var/run/docker.sock:/var/run/docker.sock
user: "1000:1000" #included user field to now update container user to admin host.
```

**2. Mounted volumes may still have root modify and exe only permission:**
* Check the mounted volumes permission
```
ls -l /mount/NAS_Server/Download
```
If your user/admin name doesnt appear and **root** is present you will need to use the following CLI and include only the affected paths mounted to the container

```
~$sudo chown -R 1000:1000 /mount/NAS_Server/Download # Change ownership to linux user with user ID 1000 for that dir

~$sudo chmod -R 775 /mount/NAS_Server/Download #ensure access permission granted to the dir
```
</details>

<details>
  <summary>Step 5 - Security</summary>

**5.1 Firewall setup**

Firewall Configuration
Install ufw (Uncomplicated Firewall):
```
sudo apt update
sudo apt install ufw
```

Allow Only Required Ports:
```
sudo ufw allow 22/tcp   # For SSH
sudo ufw allow 80/tcp # For HTTP
sudo ufw allow 443/tcp # For HTTPS
sudo ufw allow 8096/tcp # Jellyfin
sudo ufw allow 9696/tcp # prowlarr
sudo ufw allow 8989/tcp # radarr
sudo ufw allow 7878/tcp # sonarr
sudo ufw allow 7575/tcp # homarr
sudo ufw allow 8080/tcp # qBittorrent
sudo ufw allow 6881/udp # qBittorrent
sudo ufw allow 1194/udp # OpenVPN
sudo ufw enable
```

Check Firewall Status:
```
sudo ufw status
```


**5.2 fail2ban setup (WIP)**


**5.3 nginx and certbot setup (WIP)**



</details>

<details>
  <summary>Step 6 - Remote Access (WIP)</summary>
* nginx 
* certbot
* duckdns
</details>