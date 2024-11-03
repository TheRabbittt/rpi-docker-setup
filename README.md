# <img src="https://github.com/user-attachments/assets/39d7950d-8c68-4845-a20c-97ba42d940cd" width="60" alt="logo"> Raspberry-Pi Setup

This repository documents my Raspberry Pi server setup, covering configuration steps, Docker Compose files, and other useful resources. 

#### Here's a list of services with links to their repositories:
* [Nginx Proxy Manager](https://github.com/Nginxproxymanager/Nginx-Proxy-Manager)
* [Portainer](https://github.com/Portainer/Portainer)
* [Pihole](https://github.com/Pi-Hole/Pi-Hole)
  * [Cloudflared](https://github.com/Cloudflare/Cloudflared)
* [Vaultwarden](https://github.com/Dani-Garcia/Vaultwarden)
* [WireGuard](https://www.wireguard.com/)
* [Watchtower](https://github.com/Containrrr/Watchtower)
* [Filebrowser](https://github.com/hurlenko/filebrowser-docker)
---

## Prerequisites
* Adequate storage for your needs. (Example: a 128GB microSD card with 8GB used for the services below.)
* A Raspberry Pi with an installed and updated OS. (I used [Raspberry Pi OS](https://www.raspberrypi.com/software/) 64-bit, based on Debian, but any OS should work as long as you know the commands for it.)
* Static IP: Recommended for consistent access.


## Installing Docker

To get started, install ```docker``` and ```docker-compose```.

Docker is a tool that simplifies application deployment in lightweight containers. Containers share the same OS resources, so they’re more efficient than virtual machines.

Docker Compose is a tool that simplifies the setup of multiple Docker containers through YAML configuration files.

    Note: Installation varies by OS. Refer to [Docker's Official Site](https://docs.docker.com/desktop/install/debian/) for detailed instructions on your specific OS.
#### Docker on Debian 
``` Bash
curl -sSl https://get.docker.com | sh
```
To avoid running Docker commands as root, add your user to the Docker group:
``` Bash
sudo usermod -aG docker ${whoami}
```
You may need to log out and back in for this to take effect.

Verify that Docker is installed by running your first container:
``` Bash
sudo docker run hello-world
```
Install docker compose:
``` Bash
sudo apt-get install docker-compose-plugin
```
Verify that Docker Compose is installed:
``` Bash
docker compose version
```

## NGINX Proxy Manager

NGINX Proxy Manager lets you manage domains and control which application each domain points to. For example, you can create a domain name for Pi-hole (e.g., pihole.website.io) instead of using its IP address. This setup won’t be public; only devices connected to NGINX Proxy Manager will access these domains.

    Tutorial: [Video Guide](https://www.youtube.com/watch?v=qlcVx-k-02E) (I used deSEC for DNS, which is free.)

``` Bash
services:
  nginx_proxy_manager:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx_proxy_manager
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./config.json:/app/config/production.json
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    restart: always
    networks:
      proxy:
        ipv4_address: 172.19.0.10

networks:
  name: proxy
  proxy:
    ipam:
      config:
        - subnet: 172.19.0.0/16
```


Run NGINX Proxy Manager with the following Docker Compose configuration:
``` Bash
docker compose up -d
```
Access NGINX Proxy Manager at {raspberrypiIP}:81.

## Portainer
Portainer is a GUI tool for managing Docker containers.

To organize, I create a directory for each service/application and place the corresponding Docker Compose file in that directory.

##### Portainer Compose File
``` Bash
services:
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:latest
    ports:
      - 9443:9443
    volumes:
      - data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    networks:
      portainer:
        ipv4_address: 172.18.0.2
      proxy:
        ipv4_address: 172.19.0.11
volumes:
  data:

networks:
  name: portainer
  portainer:
    ipam:
      config:
        - subnet: 172.18.0.0/16
  proxy:
    external: true

```
Run Portainer with:
``` Bash
docker compose up -d
```
Access Portainer at https://{raspberrypiIP}:9443.

## Pi-Hole + Cloudflared

Pi-Hole acts as a [DNS sinkhole](https://en.wikipedia.org/wiki/DNS_sinkhole), blocking ads and telemetry requests. Paired with Cloudflared, it allows DNS over HTTPS (DoH), encrypting DNS queries.

``` Bash
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8061:80/tcp"
      - "4443:443/tcp"
    environment:
      TZ: 'Europe/Stockholm' #change this
      WEBPASSWORD: 'password' #change this
    volumes:
       - './data/etc:/etc/pihole/'
       - './data/dnsmasq.d/:/etc/dnsmasq.d/'
    restart: unless-stopped
    networks:
      pihole:
        ipv4_address: 172.30.0.2
      nginx-proxy-manager_proxy:
        ipv4_address: 172.19.0.16
    dns:
      - 127.0.0.1 #set dns within container to 127.0.0.1

  cloudflared-cf:
    container_name: cloudflared-cf
    image: cloudflare/cloudflared:latest
    command: proxy-dns --address 0.0.0.0 --port 5353 --upstream https://1.1.1.1/dns-query --upstream https://1.0.0.1/dns-query
    restart: unless-stopped
    networks:
      pihole:
        ipv4_address: 172.30.1.1

  cloudflared-goog:
    container_name: cloudflared-goog
    image: cloudflare/cloudflared:latest
    command: proxy-dns --address 0.0.0.0 --port 5353 --upstream https://8.8.8.8/dns-query --upstream https://8.8.4.4/dns-query
    restart: unless-stopped
    networks:
      pihole:
        ipv4_address: 172.30.8.8

networks:
  pihole:
    name: pihole
    ipam:
      config:
        - subnet: 172.30.0.0/16
  nginx-proxy-manager_proxy:
    external: true

```
After running the docker compose yml you should be able to reach pihole through http://{raspberrypi_ip}:8061/admin. Login password should be "changeme" although you should change the password which you can do by going into the docker container.

``` Bash
docker exec -it <container_id> bash
pihole -a -p <password>
```

For the finale configuration, go into settings in pihole and change the upstream DNS to the docker container IP addresses of cloudflared. Now that should be it for the raspberry pi, change your DNS server (typically your router) to point to the raspberry pi and boom.... you are done.

## Bitwarden/Vaultwarden
Bitwarden is a password manager and vaultwarden is a more lightweight option that you can host yourself. This works with the bitwarden app and extension. 

``` Bash
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - WEBSOCKET_ENABLED=true               
    volumes:
      - ./data:/data
    ports:
      - 8080:80  
    networks:
      bitwarden:
        ipv4_address: 172.21.0.2
      proxy:
        ipv4_address: 172.19.0.13

networks:
  name:bitwarden
  bitwarden:
    ipam:
      config:
        - subnet: 172.21.0.0/16
  proxy:
     external: true                               
```

Once the container is up you should be able to reach bitwarden through http://{raspberrypiIP}:8080, although you won't be able to create an account or use it just yet. Bitwarden needs to go through HTTPS otherwise errors will occur. There are multiple ways of doing this, one way is through a reverse proxy which I found to be the easiest. 

## Wireguard VPN
I also have a VPN on my Pi to be able to reach my DNS and my LAN in general from outside my network. There are different options out there but I choose wireguard and found it simple to configure.
``` Bash
services:
  wireguard:
    image: linuxserver/wireguard
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Stockholm                                    # Change this
      - SERVERURL=auto #optional                               # Set to automatically find server's external IP   
      - SERVERPORT=51820 #optional
      - PEERS=1 #optional                                      # Change this to the number of clients needed
      - PEERDNS=172.19.0.16 #optional                          # Pi-Hole docker IP Address (most likely a 172.... address)
      - INTERNAL_SUBNET=10.13.13.0 #optional
      - ALLOWEDIPS=0.0.0.0/0 #optional
    volumes:
      - /home/pi/wireguard/config:/config                      
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp                                        # Forward port 51820/udp on your router to the server IP
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv4.ip_forward=1
    restart: unless-stopped
    networks:
      wireguard:
        ipv4_address: 172.22.0.2
      proxy:
        ipv4_address: 172.19.0.15

networks:
  name: wireguard
  wireguard:
    ipam:
      config:
        - subnet: 172.22.0.0/16
  proxy:
     external: true
```
The server side VPN is created, for the client side run the command below to get a QR code of the configuration for the client.
``` Bash
docker exec -it wireguard /app/show-peer {peer number or name}
```
To add more clients in the future edit the peers variable in the docker-compose file and recreate the container.
## Watchtower
Watchtower automatically updates running containers.
``` Bash
docker logs watchtower
```

``` Bash
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    environment:
      TZ: Europe/Stockholm                                                                               
      WATCHTOWER_ROLLING_RESTART: 'true'
      #WATCHTOWER_MONITOR_ONLY: 'true'
      WATCHTOWER_SCHEDULE: '0 0 0 * * 0' #Runs Once A Week (Cron Epression)
      WATCHTOWER_CLEANUP: 'true'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    networks:
      watchtower:
        ipv4_address: 172.23.0.2

networks:
  name: watchtower
  watchtower:
    ipam:
      config:
        - subnet: 172.23.0.0/16
```

## Filebrowser
Filebrowser is a simple file management tool.
```Bash
services:
  filebrowser:
    image: hurlenko/filebrowser
    container_name: filebrowser
    user: "${UID}:${GID}"
    ports:
      - 444:8080
    volumes:
      - ./DATA_DIR:/data
      - ./CONFIG_DIR:/config
    environment:
      - FB_BASEURL=/filebrowser
    networks:
      filebrowser:
        ipv4_address: 172.25.0.2
      proxy:
        ipv4_address: 172.19.0.15
    restart: unless-stopped

networks:
  name: filebrowser
  filebrowser:
    ipam:
      config:
        - subnet: 172.25.0.0/16
  proxy:
     external: true

```
## Receive Discord Alerts for Raspberry Pi Overheating

First create a new server in Discord where you will get your alerts. Find the webhook link for that server and keep the link somewhere for now.

On the raspberry pi create a file, you can call it anything, I named it cpu_temp.sh. Paste what's below into it and place the discord webhook link at the correct variable in the script:

NOTE: Credit to Dave McKay for the script and tutorial on how to do it [LINK](https://www.howtogeek.com/discord-slack-alert-raspberry-pi-too-hot/)

``` Bash
#!/bin/bash

# get CPU temperature in Celsius
pi_temp=$(vcgencmd measure_temp | awk -F "[=']" '{print($2)}')

# for Fahrenheit temperatures, use this line instead
# pi_temp=$(vcgencmd measure_temp | awk -F "[=']" '{print($2 * 1.8)+32}')

# round down to an integer value
pi_temp=$(echo $pi_temp | awk -F "[.]" '{print($1)}')

# get the hostname, so we know which Pi is sending the alert
this_pi=$(hostname)

discord_pi_webhook="Discord Webhook Link"

if [[ "$pi_temp" -ge 45 ]]; then
  curl -H "Content-Type: application/json" -X POST -d '{"content":"'"ALERT! ${this_pi} CPU temp is: ${pi_temp}"'"}' $discord_pi_webhook
fi
```

To test it and make sure it's working change the 45 in the if statement to something lower like 20. Run the file ./cpu_temp.sh and you should get a notification in Discord. 

Now to automate this I used systemd timers and used this as a reference on what to do [LINK](https://www.howtogeek.com/replace-cron-jobs-with-systemd-timers/).

Create a pialert.service and pialert.timer file at /etc/systemd/system

pialert.service

``` Bash
Description="Runs pi alert script"
Requires=cpu_temp.sh

[Service]
Type=simple
ExecStart=/home/admin/pi-alert/cpu_temp.sh
User=admin #change this to your user
```

pialert.timer

``` Bash
[Unit]
Description="Timer for the pialert.service"

[Timer]
Unit=pialert.service
OnBootSec=5min
OnUnitActiveSec=10min #how often it should run

[Install]
WantedBy=timers.target
```
