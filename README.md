# Jellyfin-project
Media Server implementation to practice system administration and solution architect roles. 

## Purpose 
Purpose of this project is to provide a 'One For All' streaming service for my family and friends to access anywhere in the world! 

Secondly, the objective of this readme is to provide a step-by-step instructions to replicate the deployment of the infrastructure and used as a guide for troubleshooting.

---

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
* Bazarr - Subtitle tracker and downloader
* qBittorrent - access to ocean feature
* Homarr - GUI dashboard for monitoring Docker containers
* OpenVPN - Secure private connection between containers and internet traffic
* flaresolverr - Helps Prowlarr bypass Cloudflare protection

---

## Current Roadblock
Container config issues with deployment for:
* nginx - mime.types syntax issue
* Certbot - [Errno 13] Permission denied: '/var/log/letsencrypt/.certbot.lock

**Current status of project**: 
* Able to stream within local network only, until nginx and certbot issue is resolved. 
* 2026 - Added Bazarr to automate ENG subtitles for all movies & shows.
* 2026 Mar- Added Tailscale to allow remote access to jellyfin server from anywhere in the world.

---

## Lets get started!

**NOTE** - Please change your naming of files and directories for security reasons. I have set generic pathing to be used as examples only.

<details>
<summary>Step 1 - Setting up Docker</summary>

**1.1 Update package list**
```
sudo apt update
```

**1.2 Install prerequisites**
```
sudo apt install -y ca-certificates curl gnupg
```

**1.3 Add Docker GPG key**
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

**1.4 Setup Docker repo**
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**1.5 Install Docker engine**
```
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**1.6 Verify installation**
```
docker --version
```

**1.7 Pull required images**
```
docker pull jellyfin/jellyfin:latest
docker pull lscr.io/linuxserver/prowlarr:latest
docker pull lscr.io/linuxserver/sonarr:latest
docker pull lscr.io/linuxserver/radarr:latest
docker pull lscr.io/linuxserver/qbittorrent:latest
docker pull kylemanna/openvpn-client:latest
```

**1.8 Confirm images**
```
docker images
```
</details>

<details>
<summary>Step 2 - Mount Synology NAS to Server</summary>

**2.1 Install cifs-utils**
```
sudo apt install -y cifs-utils
```

**2.2 Create mount points**
```
sudo mkdir -p /mount/NAS_Server/Movies
sudo mkdir -p /mount/NAS_Server/Shows
sudo mkdir -p /mount/NAS_Server/Download
```

**2.3 Mount NAS**
**NOTE**<br>
NAS_IP= your external storage device private IPv4
your_username & your_password = local user/admin credentials of the storage device
```
sudo mount -t cifs //NAS_IP/NAS_Server/Movies /mount/NAS_Server/Movies -o username=your_username,password=your_password
sudo mount -t cifs //NAS_IP/NAS_Server/Shows /mount/NAS_Server/Shows -o username=your_username,password=your_password
sudo mount -t cifs //NAS_IP/NAS_Server/Download /mount/NAS_Server/Download -o username=your_username,password=your_password
```

**2.4 Verify mounts**
```
df -h
```

**2.5 Persist mounts**<br>
edit the 'fstab.bak' file to automatically mount the folders after a reboot of the server. 
```
sudo cp /etc/fstab /etc/fstab.bak
sudo nano /etc/fstab
```

Add within fstab file:<br>
**NOTE** your_username & your_password = relates to the user credential of the storage device

```
//NAS_IP/NAS_Server/Movies /mount/NAS_Server/Movies cifs username=your_username,password=your_password,iocharset=utf8 0 0
//NAS_IP/NAS_Server/Shows /mount/NAS_Server/Shows cifs username=your_username,password=your_password,iocharset=utf8 0 0
//NAS_IP/NAS_Server/Download /mount/NAS_Server/Download cifs username=your_username,password=your_password,iocharset=utf8 0 0
```

**2.6 Apply mounts**
```
sudo mount -a
```

**2.7 Check mounts**
```
ls /mount/NAS_Server/Movies
ls /mount/NAS_Server/Shows
ls /mount/NAS_Server/Download
```
**2.8 SET Permissions**
```
sudo chmod -R 755 /mount/NAS_Server
sudo chown -R $USER:$USER /mount/NAS_Server
```
</details>
<details>

<summary> Step 3 - Deploy Jellyfin Pipeline Containers </summary><br>

We will use a docker-compose.yml file to simplify the deployment and management of all containers. 
Using Compose ensures that all container settings, volumes, and dependencies are centrally managed.

**3.1. Create container directory and open dir**
```
sudo mkdir -p  ~/container-pipeline/ && cd ~/container-pipeline/
```

**3.2. Install docker-compose**
```
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

#Set executable Permission
sudo chmod +x /usr/local/bin/docker-compose   

#Verify install
docker-compose --version                      

#pull required images
sudo docker-compose pull                      

```

**3.3. Create docker-compose file**<br>
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
      - ~/jellyfin-pipeline/jellyfin-config:/config   # <-- To persist user details
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
      
  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    ports:
      - "6767:6767"
    volumes:
      - ~/jellyfin-pipeline/bazarr:/config
      - /mnt/NAS_Server/Shows:/shows
      - /mnt/NAS_Server/Movies:/movies
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Australia/Sydney
    
    flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    ports:
      - "8191:8191"
    environment:
      - LOG_LEVEL=info
      - LOG_HTML=false
      - CAPTCHA_SOLVER=none
      - TZ=Australia/Sydney
    restart: unless-stopped
```

**3.4. Deploying the containers**
```
sudo docker-compose up -d 
```

**3.4. Deploying a certain container (useful when adding new apps to the pipeline such as Bazarr)**<br> 
NOTE: pull in docker means to update and download the latest version of the container image.
```
docker compose pull bazarr 
docker compose up -d bazarr
```

**Some tips for troubleshooting:**<br>
```
#Check on status of containers
sudo docker ps

#Check logs of a container
sudo docker logs container_name
```

**3.5. Permission for containers to read, write, and execute**<br>
**Container Default User** is '1000:1000' 
**775** - 7-owner, 7-groups, 5-others, where **7**=can read, write, & exe while **5**=read and exe only
```
sudo chown -R 1000:1000 /mount/NAS_Server/Movies /mount/NAS_Server/Shows /mount/NAS_Server/Download

sudo chmod -R 775 /mount/NAS_Server/Movies /mount/NAS_Server/Shows /mount/NAS_Server/Download

```
</details>

<details>
  <summary>Step 4 - Setting up Jellyfin, Prowlarr, Sonarr, Radarr, & Bazarr</summary><br>

**4.1 Set up qBittorrent** <br>

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

**4.3.1 Linking with Radarr, Sonarr**<br>
Settings -> Select Apps -> Add (Radarr and Sonarr) -> fill the following fields:
* Authentication : Forms (login page)
* Server address = url with the port eg. 192.168.1.2:8989 (Radarr).
* API keys= API keys can be found in settings/General in the respective apps.
* Bazarr (link to radarr and sonarr)= Add API keys from Radarr and sonarr into Bazarr settings to establish connection. More info see: https://wiki.bazarr.media/Getting-Started/First-time-installation-configuration/
* Add flaresolverr in Prowlarr - Settings ->Indexers -> '+' -> flaresolverr. Set Tag field = 'cloudflare' and Host = 'http://flaresolverr:8191'. Note you will need to add the tag 'cloudflare' to indexers that has cloudflare protectiont to help bypass.

**4.3.2 Linking to qbittorrent**<br>
Settings/Download Clients -> Select qbittorrent -> include the following:
* Host = linux server
* Port = default is 8080 
* Username = qbittorrent user 
* Password = qbittorrent password

**4.4 Setting up Radarr and Sonarr**<br>
Both services' setup will be identical so I will only focus on Sonarr here.
```
http://<server_IP>:7878 # Radarr
http://<server_IP>:8989 # Sonarr
```

**4.4.1 Adding Root folder and setup login page**<br>
Within Settings/Media Management -> Root folders (add Root Folder) → in the main directory opened, select /movies.
General settings - within the Authentication field select Forms (login page).

**4.4.2 Adding Download Clients (qBittorrent)**<br>
Under Download Clients select the add icon -> select qBittorrent -> fill the following fields:
* Host = linux server
* Port = default is 8080 
* Username = qbittorrent user 
* Password = qbittorrent password
* Hit the test button and then Save. 
**Note**: Any error displayed will indicate which field(s) you have incorrectly filled out and will be useful for troubleshooting. 

**Troubleshooting**<br>
If there are permission issues with accessing root folders, try the following:<br>
**1. Does the container user have same permission as host user?**<br>
* open and edit your Docker-compose.yml file.
* Within the service sector **add user : "1000:1000"** to set container user to host admin and match same permissions to read, write or execute.
**NOTE** - defualt user ID is 1000 (most admins).<br>
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
  <summary>Step 5 - Updating Container Images</summary><br>

**5.1 Pulling in updates for Containers**<br>
Overtime the software will require updates, apply the following command to download the update for the container without stopping it:

```
cd ~/container-pipeline
sudo docker-compose pull radarr sonarr prowlarr jellyfin

```

**5.2 Restarting with the updated image**

Restart the containers to apply the latest image version(this will stop affected services):
```
sudo docker-compose up -d radarr sonarr prowlarr jellyfin
docker ps
```

**5.3 Watchtower Setup for automation (Optional & WIP):**

WatchTower - Checks for updated docker images, gracefully shuts down running containers and restarts them while preserving all data and config.
Apply the following to your docker-compose file:
```
  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TZ=Australia/Sydney
      - WATCHTOWER_POLL_INTERVAL=21600  # Checks every 6 hours (21600 seconds)/change this to your preference
      - WATCHTOWER_CLEANUP=true         # Auto remove old images
```
*NOTE:*
* docker.sock: Lets Watchtower control other containers
* WATCHTOWER_POLL_INTERVAL: Frequency it checks for updates
* WATCHTOWER_CLEANUP: Automatically removes old versions after updating
* TZ: Ensures logs use your local time

**Execute Watchtower container**
```
cd ~/container-pipeline
sudo docker-compose up -d watchtower
```
**Optional Commands:**
```
sudo docker exec watchtower watchtower --run-once #To test first before committing. 
sudo docker logs -f watchtower #To view log files from this container.
```
**5.4 Mount missing Jellyfin docker due to NAS reboots (power outage experience):**

WatchTower - On power outage from natural cause or NBN technicans, I have to manually mount my NAS drives back to the server.
Apply the following in your jellyfin-pipeline folder:
```
ping 192.168.1.100                   # NAS_IPv4 
df -h | grep Media                   # Verify if the mounts are present on your server
cat /etc/fstab | grep -i media       # Verify if fstab config file is presesnt
sudo mkdir -p /mnt/NAS_Server/Movies /mnt/NAS_Server/Shows /mnt/NAS_Server/Download      # creates mount points that has been mapped in docker-compose
sudo mount -a                        # Mounts all drives defined in /fstab
df -h | grep Media                   # To verify mounts are present
docker compose restart jellyfin      # Restart your jellyfin container which should now have access to the Media files in your NAS
```
*NOTE:*
* docker.sock: Lets Watchtower control other containers

</details>

<details>
  <summary>Step 6 - Remote Access</summary><br>

**6.1 Why Tailscale?**

*What is Tailscale?*<br>
It's a private VPN that connects your devices into one network for you to acess them anywhere without exposing them to the public internet. This is possible via an encrypted tunnel between your device and your hosted server.

*How does Tailscale work?*<br>
When you install Tailscale on a nominated device:
1.Your device joins a private network (called a tailnet).
2.Each added device gets a private IP address (like 100.x.x.10).
3.Devices can talk to each other directly and securely.

Benefits:
* No port forwarding required
* Secure access from anywhere
* Devices must be authenticated to connect
* Works across mobile and desktop

---

**6.2 Install Tailscale on Linux Server**

Install Tailscale:
```
curl -fsSL https://tailscale.com/install.sh| sh
```

Start and authenticate:
```
sudo tailscale up
```
* Open the login URL shown in terminal
* Sign in and approve the device

To Verify if the VPN is up:
```
tailscale status
tailscale ip -4
```
On the Tailscale console (web app) - Add all your devices you would like to the network. 
* You can send invites to other users to join the network.<br>
**Important note** - You should set a user role to limit access to only the jellyfin server and admin roles for other users.

---

**6.3 Accessing Jellyfin**

Once connected to Tailscale, access Jellyfin via:
```
http://YOUR_TAILSCALE_IP:[JellyfinPort] # Example 127.0.0.1:8096
```
---

**6.4 Security Notes**
* No services are exposed publicly
* Only devices in your Tailscale network can connect
* Enable 2FA on your Tailscale account
* Regularly review connected devices via logs
* Set Access controls for all users.

**Happy streaming** 📺✈️💻
</details>

<details>
  <summary>Step 7 - Automation</summary><br>

**7.1 Why Jellyseerr?**

*What is Jellyseerr?*<br>
Jellyseerr is a request management app that lets users search and request movies or TV shows from a simple web interface. It connects with Sonarr and Radarr to automate the process of adding requested content into your Jellyfin pipeline.

*Why use Jellyseerr?*<br>
Without Jellyseerr, you would need to manually add movies and shows directly into Radarr or Sonarr. Jellyseerr makes this easier by giving both you and other users a more friendly way to request content.

Benefits:
* Simple request interface for movies and TV shows
* Reduces manual work in Sonarr and Radarr
* Easier for non-technical users to use
* Can be linked with Jellyfin user accounts
* Keeps requests organised in one place

---

**7.2 Add Jellyseerr to Docker Compose**

Add the following container to your `docker-compose.yml`:

```yaml
  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    ports:
      - "5055:5055"
    environment:
      - LOG_LEVEL=debug
      - TZ=Australia/[city]
    volumes:
      - ~/jellyfin-pipeline/jellyseerr:/app/config
    restart: unless-stopped
#
Start the container:
docker compose up -d
docker ps
'''

**WIP**
---

</details>