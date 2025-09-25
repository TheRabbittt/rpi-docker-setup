# <img src="https://github.com/user-attachments/assets/39d7950d-8c68-4845-a20c-97ba42d940cd" width="60" alt="logo"> Raspberry-Pi Setup

This repository documents my raspberry pi server setup, covering configuration steps, docker compose files, and other useful resources. 

#### Here's a list of services with links to their repositories:
* [Nginx Proxy Manager](https://github.com/Nginxproxymanager/Nginx-Proxy-Manager)
* [Portainer](https://github.com/Portainer/Portainer)
* [Pihole](https://github.com/Pi-Hole/Pi-Hole)
  * [Cloudflared](https://github.com/Cloudflare/Cloudflared)
* [Vaultwarden](https://github.com/Dani-Garcia/Vaultwarden)
* [WireGuard](https://www.wireguard.com/)
* [Watchtower](https://github.com/Containrrr/Watchtower)
* [Filebrowser](https://github.com/hurlenko/filebrowser-docker)
* [Obsidian-LiveSync](https://github.com/vrtmrz/obsidian-livesync)
* [Grafana/Pi Monitoring](https://github.com/oijkn/Docker-Raspberry-PI-Monitoring?tab=readme-ov-file)
* [*arr Stack]()
  * [Overseerr](https://github.com/sct/overseerr)
  * [Radarr](https://github.com/Radarr/Radarr)
  * [Sonarr](https://github.com/Sonarr/Sonarr)
  * [Prowlarr](https://github.com/Prowlarr/Prowlarr)
  * [Flaresolverr](https://github.com/FlareSolverr/FlareSolverr)
  * [qBittorrent](https://github.com/linuxserver/docker-qbittorrent)
---

## Prerequisites
* Adequate storage for your needs. (Example: I used a 128GB microSD card with 20% used for all my services.)
* A Raspberry Pi with an installed and updated OS. (I used [Raspberry Pi OS](https://www.raspberrypi.com/software/) 64-bit, based on Debian, but any OS should work as long as you know the commands for it.)
* Static IP: Recommended for consistent access.


## Installing Docker

To get started, install ```docker``` and ```docker-compose```.

Docker is a tool that simplifies application deployment in lightweight containers. Containers share the same OS resources, so they’re more efficient than virtual machines.

Docker compose is a tool that simplifies the setup of multiple docker containers through YAML configuration files.

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

NGINX Proxy Manager lets you manage domains and control which application each domain points to. For example, you can create a domain name for Pi-hole (e.g., pihole.website.io) instead of using its IP address. This setup won’t be public; only devices within your LAN will be able to access these services.

   Tutorial: Credit to "Wolfgang's Channel" on youtube for this [video](https://www.youtube.com/watch?v=qlcVx-k-02E) guide (I used deSEC for DNS, which is free.)

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
        ipv4_address: 172.20.0.2

networks:
  proxy:
    name: proxy
    ipam:
      config:
        - subnet: 172.20.0.0/16
```


Run NGINX Proxy Manager with the following Docker Compose configuration:
``` Bash
docker compose up -d
```
Access NGINX Proxy Manager at ```http://{raspberrypi-ip}:81```.

## Portainer
Portainer is a GUI tool for managing Docker containers.

To organize my docker compose files, I create a directory for each service/application and place the corresponding docker compose file in that directory.

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
    networks:
      portainer:
        ipv4_address: 172.30.0.2
      proxy:
        ipv4_address: 172.20.0.3
    restart: unless-stopped
  
volumes:
  data:
  
networks:
  portainer:
    name: portainer
    ipam:
      config:
        - subnet: 172.30.0.0/16
  proxy:
     external: true
```
Run Portainer with:
``` Bash
docker compose up -d
```
Access Portainer at ```https://{raspberrypi-ip}:9443```.

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
      TZ: 'Europe/Stockholm'                  #change this
      WEBPASSWORD: password                   #change this
    volumes:
       - './data/etc:/etc/pihole/'
    restart: unless-stopped
    networks:
      pihole:
        ipv4_address: 172.50.0.2
      proxy:
        ipv4_address: 172.20.0.4

  cloudflared-cf:
    container_name: cloudflared-cf
    image: cloudflare/cloudflared:latest
    command: proxy-dns --address 0.0.0.0 --port 5353 --upstream https://1.1.1.1/dns-query --upstream https://1.0.0.1/dns-query
    restart: unless-stopped
    networks:
      pihole:
        ipv4_address: 172.50.1.1

  cloudflared-goog:
    container_name: cloudflared-goog
    image: cloudflare/cloudflared:latest
    command: proxy-dns --address 0.0.0.0 --port 5353 --upstream https://8.8.8.8/dns-query --upstream https://8.8.4.4/dns-query
    restart: unless-stopped
    networks:
      pihole:
        ipv4_address: 172.50.8.8

networks:
  pihole:
    name: pihole
    ipam:
      config:
        - subnet: 172.50.0.0/16
  proxy:
    external: true
```
After running the docker compose yml you should be able to reach pihole through ```http://{raspberrypi_ip}:8061/admin```. Login password should be "changeme" although you should change the password which you can do by going into the docker container.

``` Bash
docker exec -it <container_id> bash
pihole -a -p <password>
```

For the finale configuration, go into settings in pihole and change the upstream DNS to the docker container IP addresses of cloudflared (172.30.1.1 & 172.30.8.8). Now that should be it for the raspberry pi, change your DNS server (typically your router) to point to the raspberry pi and boom.... you are done.

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
        ipv4_address: 172.70.0.2
      proxy:
        ipv4_address: 172.20.0.5

networks:
  bitwarden:
    name: bitwarden
    ipam:
      config:
        - subnet: 172.70.0.0/16
  proxy:
    external: true                             
```

Once the container is up you should be able to reach bitwarden through ```http://{raspberrypi-ip}:8080```, although you won't be able to create an account or use it just yet. Bitwarden needs to go through HTTPS otherwise errors will occur. There are multiple ways of doing this, one way is through a reverse proxy which I found to be the easiest. 

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
      - PEERS=3 #optional                                      # Change this to the number of clients needed
      - PEERDNS=172.20.0.4  #optional                          # Pi-Hole docker IP Address (most likely a 172.... address)
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
        ipv4_address: 172.40.0.2
      proxy:
        ipv4_address: 172.20.0.7

networks:
  wireguard:
    name: wireguard
    ipam:
      config:
        - subnet: 172.40.0.0/16
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
      WATCHTOWER_SCHEDULE: '0 0 0 * * 0'          #Runs Once A Week (Cron Epression)
      WATCHTOWER_CLEANUP: 'true'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    networks:
      watchtower:
        ipv4_address: 172.120.0.2

networks:
  watchtower:
    name: watchtower
    ipam:
      config:
        - subnet: 172.120.0.0/16
```

## Filebrowser
Filebrowser is a simple file management tool.
```Bash
services:
  filebrowser:
    container_name: filebrowser
    image: hurlenko/filebrowser
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
        ipv4_address: 172.110.0.2
      proxy:
        ipv4_address: 172.20.0.6
    restart: unless-stopped

networks:
  filebrowser:
    name: filebrowser
    ipam:
      config:
        - subnet: 172.110.0.0/16
  proxy:
     external: true
```

## Obsidian-LiveSync

Obsidian is a note-taking app that uses plain-text Markdown files stored locally. Obsidian LiveSync is a self-hosted synchronization plugin you can run on a Raspberry Pi, enabling real-time, end-to-end encrypted syncing of your notes across multiple devices without relying on third-party cloud services.

```Bash
services:
   couchdb-obsidian-livesync:
    container_name: obsidian-livesync
    image: couchdb:latest
    environment:
      - TZ=Europe/Stockholm         #change this
      - COUCHDB_USER=admin          #change this
      - COUCHDB_PASSWORD=password   #change this
    volumes:
      - ./data:/opt/couchdb/data
      - ./etc:/opt/couchdb/etc/local.d
    ports:
      - "5984:5984"
    networks:
      obsidian:
        ipv4_address: 172.60.0.2
    restart: unless-stopped
  
networks:
  obsidian:
    name: obsidian
    ipam:
      config:
        - subnet: 172.60.0.0/16
```

Next you will have to setup the database, I would recommend following this [Guide](https://www.reddit.com/r/selfhosted/comments/1eo7knj/guide_obsidian_with_free_selfhosted_instant_sync/)

## *arr

For my arr stack I run everything within the same docker compose configuration file. Seems logical since they are mostly dependent on eachother. The initial setup of all of these is pretty simple but if you have trouble or things you would like to optimize I would recommend using this guide [Trash Guide](https://trash-guides.info/).

My docker compose stack is a little unique. I have mounted my NAS drive to my raspberry pi. You will have to change volumes specifically /mnt/BigBoi/x:/data part.

``` Bash
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1026
      - PGID=1000
      - TZ=Europe/Stockholm                                 #change this
    volumes:
      - radarr_config:/config
      - /mnt/BigBoi/data:/data                              #change this, in this example I have mounted my NAS share to my raspberry pi.
    ports:
      - 7878:7878
    restart: unless-stopped
    networks:
      arr:
        ipv4_address: 172.100.0.2

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1026
      - PGID=1000
      - TZ=Europe/Stockholm                                 #change this
    volumes:
      - sonarr_config:/config
      - /mnt/BigBoi/data:/data                              #change this, in this example I have mounted my NAS share to my raspberry pi.
    ports:
      - 8989:8989
    restart: unless-stopped
    networks:
      arr:
        ipv4_address: 172.100.0.3
    
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1026                  
      - PGID=1000
      - TZ=Europe/Stockholm                                 #change this
      - WEBUI_PORT=8081
      - TORRENTING_PORT=6881
    volumes:
      - qbittorrent_config:/config
      - /mnt/BigBoi/data/Downloads:/data/Downloads          #change this, in this example I have mounted my NAS share to my raspberry pi.
    restart: unless-stopped
    network_mode: "container:gluetun"
    
  overseerr:
    image: lscr.io/linuxserver/overseerr:latest
    container_name: overseerr
    environment:
      - PUID=1026
      - PGID=1000
      - TZ=Europe/Stockholm                                #change this
    volumes:
      - overseerr_config:/config
    ports:
      - 5055:5055
    restart: unless-stopped
    networks:
      arr:
        ipv4_address: 172.100.0.4

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1026
      - PGID=1000
      - TZ=Europe/Stockholm                                #change this
    volumes:
      - prowlarr_config:/config
    ports:
      - 9696:9696
    restart: unless-stopped
    networks:
      arr:
        ipv4_address: 172.100.0.5

  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - LOG_HTML=${LOG_HTML:-false}
      - CAPTCHA_SOLVER=${CAPTCHA_SOLVER:-none}
      - TZ=Europe/Stockholm                                #change this
    volumes:
      - flaresolverr_config:/config
    ports:
      - "${PORT:-8191}:8191"
    restart: unless-stopped
    networks:
      arr:
        ipv4_address: 172.100.0.6

networks:
  arr:
    driver: bridge
    name: arr
    ipam:
      config:
        - subnet: 172.100.0.0/16

volumes:
  radarr_config:
  sonarr_config:
  overseerr_config:
  prowlarr_config:
  flaresolverr_config:
  qbittorrent_config:
```

## Grafana

To setup grafana I followed oijkn's guide on github  [Oijkn](https://github.com/oijkn/Docker-Raspberry-PI-Monitoring?tab=readme-ov-file)

``` Bash
services:
  grafana:
    container_name: monitoring-grafana
    image: grafana/grafana:latest
    hostname: rpi-grafana
    restart: unless-stopped
    user: "472"
    networks:
      - monitor
    ports:
      - "3000:3000"
    env_file:
      - ./grafana/.env
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus
    healthcheck:
      test: ["CMD", "wget", "-O", "/dev/null", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    labels:
      - "com.example.description=Grafana Dashboard"
      - "com.example.service=monitoring"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  cadvisor:
    container_name: monitoring-cadvisor
    image: gcr.io/cadvisor/cadvisor:latest
    hostname: rpi-cadvisor
    restart: unless-stopped
    cap_add:
      - SYS_ADMIN
    networks:
      - monitor
    expose:
      - 8080
    command:
      - '-housekeeping_interval=15s'
      - '-docker_only=true'
      - '-store_container_labels=false'
    devices:
      - /dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
      - /etc/machine-id:/etc/machine-id:ro
    healthcheck:
      test: ["CMD", "wget", "-O", "/dev/null", "http://localhost:8080/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
    labels:
      - "com.example.description=cAdvisor Container Monitoring"
      - "com.example.service=monitoring"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    mem_limit: 256m
    mem_reservation: 128m

  node-exporter:
    container_name: monitoring-node-exporter
    image: prom/node-exporter:latest
    hostname: rpi-exporter
    restart: unless-stopped
    networks:
      - monitor
    expose:
      - 9100
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --path.rootfs=/host
      - --collector.filesystem.ignored-mount-points
      - ^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($$|/)
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
      - /:/host:ro,rslave
    healthcheck:
      test: ["CMD", "wget", "-O", "/dev/null", "http://localhost:9100/metrics"]
      interval: 30s
      timeout: 10s
      retries: 3
    labels:
      - "com.example.description=Node Exporter"
      - "com.example.service=monitoring"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    mem_limit: 128m
    mem_reservation: 64m

  prometheus:
    container_name: monitoring-prometheus
    image: prom/prometheus:latest
    hostname: rpi-prometheus
    restart: unless-stopped
    user: "nobody"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=1y'
      - '--storage.tsdb.retention.size=10GB'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    networks:
      - monitor
    expose:
      - 9090
    volumes:
      - prometheus-data:/prometheus
      - ./prometheus:/etc/prometheus/
    depends_on:
      - cadvisor
      - node-exporter
    healthcheck:
      test: ["CMD", "wget", "-O", "/dev/null", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    labels:
      - "com.example.description=Prometheus Time Series Database"
      - "com.example.service=monitoring"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    mem_limit: 1g
    mem_reservation: 512m

volumes:
  grafana-data:
    labels:
      - "com.example.description=Grafana Persistent Data"
      - "com.example.service=monitoring"
  prometheus-data:
    labels:
      - "com.example.description=Prometheus Persistent Data"
      - "com.example.service=monitoring"

networks:
  monitor:
    driver: bridge
    name: grafana
    ipam:
      config:
        - subnet: 172.80.0.0/16
          gateway: 172.80.0.1
    labels:
      - "com.example.description=Monitoring Network"
      - "com.example.service=monitoring"
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

if [[ "$pi_temp" -ge 50 ]]; then
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

