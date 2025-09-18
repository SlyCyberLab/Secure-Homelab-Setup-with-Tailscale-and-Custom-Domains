# Secure Homelab Setup with Tailscale and Custom Domains

A comprehensive guide to creating a secure homelab with custom domains, automatic SSL certificates, and zero port forwarding using Tailscale, Docker, and Nginx Proxy Manager.

## Architecture Overview

This setup creates a secure homelab accessible via custom domains without exposing any ports to the internet. Services are accessible through clean URLs like `https://portainer.homelab.example.com` only to devices connected to your Tailscale network.

**Technology Stack:**
- Tailscale (Zero-trust VPN)
- Docker & Docker Compose
- Nginx Proxy Manager (Reverse Proxy)
- dnsmasq (Internal DNS)
- Let's Encrypt (SSL Certificates)

## Prerequisites

- Domain name (e.g., `example.com`)
- Cloudflare account with domain management
- Tailscale account
- Ubuntu Server 22.04+ with Docker
- Minimum 4GB RAM, 20GB storage

## Phase 1: Domain and DNS Preparation

### 1.1 Create Cloudflare API Token

1. Go to [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens)
2. Click "Create Token" → Use "Edit zone DNS" template
3. Set permissions: Zone / DNS / Edit
4. Zone Resources: Include your domain (`example.com`)
5. Save the API token for later use

### 1.2 DNS Planning

We'll use a hierarchical subdomain structure: `service.servername.example.com`

This provides better organization for multiple servers:
- `npm.docker-host.example.com` - Nginx Proxy Manager
- `portainer.docker-host.example.com` - Container Management  
- `vault.docker-host.example.com` - Password Manager

**Benefits of this structure:**
- Clear service identification
- Easy scaling to multiple hosts
- Logical grouping by server function
- Professional enterprise-like naming

## Phase 2: Tailscale Network Setup

### 2.1 Install Tailscale on Host Server

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Configure with subnet routing (adjust network range as needed)
tailscale up --advertise-routes=192.168.1.0/24 --ssh
```

### 2.2 Approve Subnet Routes

1. Go to Tailscale admin console
2. Navigate to Machines → find your host
3. Approve the advertised subnet route

### 2.3 Get Tailnet Information

```bash
# Find your tailnet name for later reference
tailscale status --json | grep MagicDNSSuffix
# Example: "example-network.ts.net"
```

## Phase 3: Docker Environment Setup

### 3.1 Install Docker

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

# Add user to docker group
sudo usermod -aG docker $USER
```

Log out and back in for group changes to take effect.

## Phase 4: Internal DNS Server

### 4.1 Deploy dnsmasq for Internal DNS

```bash
# Create DNS server directory
mkdir ~/dns-server && cd ~/dns-server

# Create docker-compose.yml
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
```

### 4.2 Configure DNS Resolution

```bash
# Create DNS configuration (adjust IP to your Docker host)
cat > dnsmasq.conf << EOF
# Route all docker-host.example.com queries to local Docker host
address=/docker-host.example.com/192.168.1.100

# Upstream DNS servers
server=8.8.8.8
server=1.1.1.1
EOF

# Start DNS server
docker compose up -d
```

### 4.3 Configure Tailscale SplitDNS

1. Go to Tailscale admin console → DNS
2. Add nameserver: `192.168.1.100` (your Docker host IP)
3. Restrict to domain: `example.com`
4. Enable "Override local DNS"

## Phase 5: Nginx Proxy Manager Deployment

### 5.1 Deploy NPM

```bash
# Create NPM directory
mkdir ~/nginx-proxy-manager && cd ~/nginx-proxy-manager

# Create docker-compose.yml
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

# Start NPM
docker compose up -d
```

### 5.2 Initial NPM Setup

1. Access NPM at `http://192.168.1.100:8181`
2. Login with default credentials:
   - Email: `admin@example.com`
   - Password: `changeme`
3. Change default password and email

### 5.3 Create Wildcard SSL Certificate

1. Go to SSL Certificates → Add SSL Certificate
2. Domain Names: `*.docker-host.example.com`
3. Use DNS Challenge → Select Cloudflare
4. Enter Cloudflare API credentials:
   ```
   dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN
   ```
5. Request certificate

## Phase 6: Service Deployment

### 6.1 Deploy Portainer

```bash
# Create Portainer directory
mkdir ~/portainer && cd ~/portainer

# Create docker-compose.yml
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

# Start Portainer
docker compose up -d
```

### 6.2 Deploy Vaultwarden

```bash
# Create Vaultwarden directory
mkdir ~/vaultwarden && cd ~/vaultwarden

# Create docker-compose.yml
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

# Start Vaultwarden
docker compose up -d
```

## Phase 7: Proxy Host Configuration

### 7.1 Create NPM Proxy Hosts

Access NPM admin interface and create proxy hosts for each service:

**NPM Admin Interface:**
- Domain Names: `npm.docker-host.example.com`
- Forward Hostname/IP: `192.168.1.100`
- Forward Port: `8181`
- SSL Certificate: `*.docker-host.example.com`
- Force SSL: Enabled

**Portainer:**
- Domain Names: `portainer.docker-host.example.com`
- Forward Hostname/IP: `192.168.1.100`
- Forward Port: `9000`
- SSL Certificate: `*.docker-host.example.com`
- Force SSL: Enabled

**Vaultwarden:**
- Domain Names: `vault.docker-host.example.com`
- Forward Hostname/IP: `192.168.1.100`
- Forward Port: `8080`
- SSL Certificate: `*.docker-host.example.com`
- Force SSL: Enabled

## Phase 8: Testing and Verification

### 8.1 DNS Resolution Test

From any Tailscale-connected device:

```bash
# Test DNS resolution
nslookup npm.docker-host.example.com
nslookup portainer.docker-host.example.com
nslookup vault.docker-host.example.com

# All should resolve to 192.168.1.100
```

### 8.2 Service Access Test

Access services via browser from Tailscale-connected devices:
- `https://npm.docker-host.example.com` - NPM admin interface
- `https://portainer.docker-host.example.com` - Container management
- `https://vault.docker-host.example.com` - Password manager

### 8.3 SSL Certificate Verification

Verify all services show valid SSL certificates with green padlock icons.

## Security Benefits

- **Zero Port Forwarding**: No ports exposed to public internet
- **Zero Trust Access**: Services only accessible via Tailscale network
- **Automatic SSL**: Valid certificates for all services
- **Internal DNS**: Custom domains work seamlessly
- **Encrypted Traffic**: All communication encrypted by Tailscale

## Troubleshooting

### DNS Issues
```bash
# Check DNS container
docker logs internal-dns

# Verify dnsmasq configuration
cat ~/dns-server/dnsmasq.conf

# Test DNS resolution locally
nslookup npm.docker-host.example.com 192.168.1.100
```

### SSL Certificate Issues
```bash
# Check NPM logs
docker logs nginx-proxy-manager

# Verify Cloudflare API token permissions
# Ensure token has Zone:DNS:Edit permissions
```

### Service Connectivity
```bash
# Check all containers are running
docker ps

# Test direct IP access
curl -I http://192.168.1.100:8181  # NPM
curl -I http://192.168.1.100:9000  # Portainer
curl -I http://192.168.1.100:8080  # Vaultwarden
```

## Adding New Services

To add additional services:

1. Deploy the new service with Docker Compose
2. Note the service's port number
3. Create new proxy host in NPM:
   - Domain: `newservice.docker-host.example.com`
   - Forward to: `192.168.1.100:[PORT]`
   - SSL Certificate: `*.docker-host.example.com`

The wildcard DNS and SSL certificate automatically handle new subdomains.

**Example for multiple servers:**
- `npm.docker-host.example.com` - Main Docker server
- `grafana.monitoring-server.example.com` - Monitoring server
- `wiki.database-server.example.com` - Database server

This structure scales well as your homelab grows.

## Architecture Advantages

- **Scalable**: Easy to add new services
- **Secure**: No external attack surface
- **Professional**: Clean domain structure
- **Automated**: SSL certificate renewal
- **Flexible**: Works from anywhere with Tailscale
- **Cost Effective**: Uses single wildcard certificate

This architecture provides enterprise-level security and functionality for homelab environments while maintaining simplicity and ease of management.
