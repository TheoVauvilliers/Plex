# Plex Ecosystem on Ubuntu 24 with Docker + Cloudflare Tunnel

This project sets up a complete Plex media server ecosystem on Ubuntu 24 using Docker containers. It includes Plex, Sonarr, Radarr, Overseerr, and torrent clients (qBittorrent), secured behind a VPN (ProtonVPN via Gluetun container) and remotely accessible via Cloudflare Tunnel, avoiding exposing your home network ports.

---

## Components

- **Plex** — Media server  
- **Sonarr** — TV show management  
- **Radarr** — Movie management  
- **Overseerr** — Requests & user interface  
- **qBittorrent** — Torrent client  
- **Gluetun** — VPN client container (ProtonVPN)  
- **Prowlarr & Byparr** — Indexers and automation  
- **Cloudflared** — Cloudflare Tunnel for secure external access  

---

## Prerequisites

- Ubuntu 24 server with Docker and Docker Compose installed  
- Domain managed by Cloudflare (e.g., `theovauvilliers.com`)  
- Plex claim token (https://www.plex.tv/claim/)  
- ProtonVPN account credentials (for VPN in Gluetun)  
- Docker environment variables set (`PUID`, `PGID`, `TZ`, etc.)  
- Media mounted at `/mnt/media/series`, `/mnt/media/movies`, and `/mnt/media/downloads`  
- Docker-compose file placed under `/opt/docker/plex/docker-compose.yml` (or your preferred directory)

---

## Setup Instructions

### 1. Prepare Environment Variables

Create a `.env` file or export environment variables for your user IDs, timezone, VPN credentials, Plex claim token, etc.

```bash
export PUID=1000
export PGID=1000
export TZ=Europe/Paris
export OPENVPN_USER=your_protonvpn_username
export OPENVPN_PASSWORD=your_protonvpn_password
export PLEX_CLAIM=your_plex_claim_token
```

### 2. Prepare Directory Structure & Permissions

Make sure directories exist and have correct permissions for Docker volumes:


```bash
mkdir -p /opt/docker/plex/{app,gluetun,qbittorrent/config,sonarr,radarr,prowlarr,overseerr}
chown -R $PUID:$PGID /opt/docker/plex
```

### 3. Docker Compose

Place your docker-compose.yml file in /opt/docker/plex/ (content as provided, with containers configured).

### 4. Setting up Cloudflare Tunnel (Cloudflared)


Install cloudflared binary on your host (optional, for tunnel setup commands):

```bash
curl -LO https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64
chmod +x cloudflared-linux-amd64
sudo mv cloudflared-linux-amd64 /usr/local/bin/cloudflared
```

**Authenticate and create tunnel**

Run these commands on your Ubuntu host (not inside Docker):

```bash
cloudflared login
# This opens a browser window to authenticate your Cloudflare account and select your domain.

cloudflared tunnel create plex-tunnel
# This generates a tunnel credentials file (~/.cloudflared/plex-tunnel.json)
```

**Move tunnel credentials and create config folder**


```bash
mkdir -p /opt/docker/plex/cloudflared
mv ~/.cloudflared/plex-tunnel.json /opt/docker/plex/cloudflared/
mv ~/.cloudflared/cert.pem /opt/docker/plex/cloudflared/
```

**Make sure these files are owned by the user running Docker:**

```bash
chown -R $PUID:$PGID /opt/docker/plex/cloudflared
chmod 600 /opt/docker/plex/cloudflared/plex-tunnel.json
chmod 600 /opt/docker/plex/cloudflared/cert.pem
```

**Create** `config.yml` **in** `/opt/docker/plex/cloudflared/`

```yml
tunnel: plex-tunnel
credentials-file: /etc/cloudflared/plex-tunnel.json

ingress:
  - hostname: plex.theovauvilliers.com
    service: http://localhost:32400

  - hostname: overseerr.theovauvilliers.com
    service: http://localhost:5055

  - service: http_status:404
```

### 5. Configure DNS in Cloudflare

- Create CNAME DNS records pointing `plex.theovauvilliers.com` and `overseerr.theovauvilliers.com` to `your_tunnel_id.cfargotunnel.com` (Cloudflare Tunnel URL) or use Cloudflare's Tunnel DNS routing command:

```bash
cloudflared tunnel route dns plex-tunnel plex.theovauvilliers.com
cloudflared tunnel route dns plex-tunnel overseerr.theovauvilliers.com
```

If you get errors about existing records, remove or update conflicting DNS entries in Cloudflare dashboard.

### 6. Start Docker stack

```bash
cd /opt/docker/plex
docker-compose up -d
```

Make sure all containers are running:

```bash
docker ps
```

### 7. Access your services externally

- Go to `https://plex.theovauvilliers.com` (Cloudflare will proxy the traffic via the tunnel to your local Plex on port 32400)
- Go to `https://overseerr.theovauvilliers.com` for Overseerr UI

### 8. Configure Overseerr

In Overseerr settings, configure the URLs for Sonarr and Radarr internally (using local IP or container IP) since they are not exposed externally.

---

### Additional Notes

- VPN: All torrent clients and indexers use Gluetun container to route traffic via ProtonVPN.
- Hardlinking: Supported inside the same filesystem. Multi-SSD setup will require special care if you want hardlinking across disks (usually impossible without a volume manager).
- Network: Cloudflared uses network_mode: host to directly access local services on the host.
- Backups: Backup your Plex config, Overseerr config, and Cloudflare tunnel credentials regularly.

---

## Rebuilding From Scratch

### 1. Remove old containers:

```bash
docker-compose down -v
```

### 2. Clean old config directories if needed (make backups first!):

```bash
rm -rf /opt/docker/plex/*
```

### 3. Recreate directory structure and `.env` file.

### 4. Re-run Cloudflare tunnel creation if needed, or reuse existing tunnel credentials.

### 5. Start containers:

```bash
docker-compose up -d
```

### 6. Recreate directory structure and `.env` file.

```bash
docker logs cloudflared -f
```

---

## Acknowledgements

A big thank you to [rickklaasboer](https://gist.github.com/rickklaasboer/b5c159833ff2971fccd32296d8ba2260) for the invaluable guide that helped shape this project.
