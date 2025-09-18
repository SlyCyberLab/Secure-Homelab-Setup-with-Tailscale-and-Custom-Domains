# Secure Homelab with Tailscale and Custom Domains

> **A zero-trust homelab setup demonstrating enterprise-level networking, automatic SSL certificates, and custom domain management without port forwarding.**

![Lab Status](https://img.shields.io/badge/Lab%20Status-Active-brightgreen)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED)
![Tailscale](https://img.shields.io/badge/Tailscale-VPN-000000)
![SSL](https://img.shields.io/badge/SSL-Let's%20Encrypt-green)

## Overview

This project transforms basic IP-based service access into a professional homelab with custom domains and enterprise-grade security. Services are accessible through clean URLs like `https://portainer.docker-host.example.com` exclusively to authorized devices on your private network.

The setup eliminates the need for traditional port forwarding while providing automatic SSL certificate management and seamless DNS resolution through Tailscale's zero-trust architecture.

**Technology Stack:**
- Tailscale (Zero-trust VPN)
- Docker & Docker Compose
- Nginx Proxy Manager (Reverse Proxy)
- dnsmasq (Internal DNS)
- Let's Encrypt (SSL Certificates)

## Prerequisites

- Domain name with Cloudflare DNS management
- Tailscale account
- Ubuntu Server with Docker installed
- Minimum 4GB RAM, 20GB storage

## Installation

### 1. Tailscale Setup

```bash
# Install Tailscale on host server
curl -fsSL https://tailscale.com/install.sh | sh

# Configure with subnet routing
tailscale up --advertise-routes=192.168.1.0/24 --ssh
```

Go to Tailscale admin console and approve the subnet route.

### 2. Docker Installation

```bash
# Remove old packages
for pkg in docker.io docker-doc docker-compose docker-compose-v2; do 
    sudo apt-get remove $pkg
done

# Add Docker repository
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

### 3. Internal DNS Server

```bash
mkdir ~/dns-server && cd ~/dns-server

cat > docker-compose.yml << EOF
services:
  dnsmasq:
    image: jpillora/dnsmasq
    container_name: internal-dns
    ports:
      - "53:53/udp"
    volumes:
      - ./dnsmasq.conf:/etc/dnsmasq.conf
    restart: unless-stopped
EOF

cat > dnsmasq.conf << EOF
address=/docker-host.example.com/192.168.1.100
server=8.8.8.8
server=1.1.1.1
EOF

docker compose up -d
```

### 4. Configure Tailscale SplitDNS

In Tailscale admin console:
- Add nameserver: `192.168.1.100`
- Restrict to domain: `example.com`
- Enable "Override local DNS"

### 5. Nginx Proxy Manager

```bash
mkdir ~/nginx-proxy-manager && cd ~/nginx-proxy-manager

cat > docker-compose.yml << EOF
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
      - '8181:81'
    volumes:
      - npm-data:/data
      - npm-ssl:/etc/letsencrypt

volumes:
  npm-data:
  npm-ssl:
EOF

docker compose up -d
```

Access NPM at `http://192.168.1.100:8181`
- Default: `admin@example.com` / `changeme`
- Change password and create wildcard SSL certificate for `*.docker-host.example.com`

### 6. Deploy Services

#### Portainer

```bash
mkdir ~/portainer && cd ~/portainer

cat > docker-compose.yml << EOF
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer-data:/data

volumes:
  portainer-data:
EOF

docker compose up -d
```

#### Vaultwarden

```bash
mkdir ~/vaultwarden && cd ~/vaultwarden

cat > docker-compose.yml << EOF
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - WEBSOCKET_ENABLED=true
    volumes:
      - vw-data:/data
    ports:
      - "8080:80"
      - "3012:3012"

volumes:
  vw-data:
EOF

docker compose up -d
```

#### Wiki.js

```bash
mkdir ~/wikijs && cd ~/wikijs

cat > docker-compose.yml << EOF
services:
  wiki:
    image: ghcr.io/requarks/wiki:2
    container_name: wikijs
    depends_on:
      - db
    environment:
      DB_TYPE: postgres
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: wikijs
      DB_PASS: wikijsrocks
      DB_NAME: wiki
    restart: unless-stopped
    ports:
      - "3000:3000"

  db:
    image: postgres:15-alpine
    container_name: wiki-db
    environment:
      POSTGRES_DB: wiki
      POSTGRES_PASSWORD: wikijsrocks
      POSTGRES_USER: wikijs
    restart: unless-stopped
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
EOF

docker compose up -d
```

#### Uptime Kuma

```bash
mkdir ~/uptime-kuma && cd ~/uptime-kuma

cat > docker-compose.yml << EOF
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    volumes:
      - uptime-kuma:/app/data
    ports:
      - "3001:3001"
    restart: unless-stopped

volumes:
  uptime-kuma:
EOF

docker compose up -d
```

#### Watchtower

```bash
mkdir ~/watchtower && cd ~/watchtower

cat > docker-compose.yml << EOF
services:
  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_POLL_INTERVAL=86400
      - WATCHTOWER_NOTIFICATIONS=shoutrrr
      - WATCHTOWER_NOTIFICATION_URL=logger://
      - WATCHTOWER_INCLUDE_RESTARTING=true
      - WATCHTOWER_ROLLING_RESTART=true
    restart: unless-stopped
EOF

docker compose up -d
```

### 7. Configure Proxy Hosts

In NPM admin interface, create proxy hosts for each service:

**NPM Admin:**
- Domain: `npm.docker-host.example.com`
- Forward: `192.168.1.100:8181`
- SSL Certificate: `*.docker-host.example.com`

**Portainer:**
- Domain: `portainer.docker-host.example.com`
- Forward: `192.168.1.100:9000`
- SSL Certificate: `*.docker-host.example.com`

**Vaultwarden:**
- Domain: `vault.docker-host.example.com`
- Forward: `192.168.1.100:8080`
- SSL Certificate: `*.docker-host.example.com`

**Wiki.js:**
- Domain: `wiki.docker-host.example.com`
- Forward: `192.168.1.100:3000`
- SSL Certificate: `*.docker-host.example.com`

**Uptime Kuma:**
- Domain: `status.docker-host.example.com`
- Forward: `192.168.1.100:3001`
- SSL Certificate: `*.docker-host.example.com`

## Testing

From any Tailscale-connected device:

```bash
# Test DNS resolution
nslookup npm.docker-host.example.com
nslookup portainer.docker-host.example.com

# Access services via browser
https://npm.docker-host.example.com
https://portainer.docker-host.example.com
https://vault.docker-host.example.com
https://wiki.docker-host.example.com
https://status.docker-host.example.com
```

## Security Features

- No exposed ports to internet
- Zero-trust access through Tailscale
- Automatic SSL certificate management
- Internal DNS resolution
- Container isolation

## Troubleshooting

**DNS not resolving:**
- Check Tailscale SplitDNS configuration
- Verify dnsmasq container is running

**SSL certificate issues:**
- Confirm Cloudflare API token permissions
- Check NPM certificate logs

**Service not accessible:**
- Verify container is running with `docker ps`
- Check NPM proxy host configuration
