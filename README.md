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

## Lets get started!
- TBA

