# Secure Homelab with Tailscale and Custom Domains

> **A zero-trust homelab setup demonstrating enterprise-level networking, automatic SSL certificates, and custom domain management without port forwarding.**

![Lab Status](https://img.shields.io/badge/Lab%20Status-Active-brightgreen)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED)
![Tailscale](https://img.shields.io/badge/Tailscale-VPN-000000)
![SSL](https://img.shields.io/badge/SSL-Let's%20Encrypt-green)

## Overview

This project transforms basic IP-based service access into a professional homelab with custom domains and enterprise-grade security. Services are accessible through clean URLs like `https://portainer.docker-host.example.com` exclusively to authorized devices on your private network.

The setup eliminates the need for traditional port forwarding while providing automatic SSL certificate management and seamless DNS resolution through Tailscale's zero-trust architecture.

### Key Features

- **Zero Port Forwarding**: No firewall holes or exposed services
- **Custom Domains**: Professional `service.server.domain.com` structure
- **Automatic SSL**: Wildcard certificates with auto-renewal
- **Internal DNS**: SplitDNS routing for seamless resolution
- **Enterprise Security**: Zero-trust network access model
- **Easy Scaling**: Add services with simple proxy host creation

## Architecture

The system uses a layered approach combining multiple technologies:

```
Internet → Tailscale Network → Internal DNS → Reverse Proxy → Services
```

**Technology Stack:**
- **Tailscale** - Zero-trust VPN with subnet routing
- **Docker & Docker Compose** - Containerized service deployment
- **Nginx Proxy Manager** - Reverse proxy with SSL termination
- **dnsmasq** - Internal DNS server for custom domains
- **Let's Encrypt** - Automated SSL certificate management
- **Cloudflare** - DNS challenge provider for wildcard certificates

## Services Deployed

### Core Infrastructure
- **Nginx Proxy Manager** - Web-based reverse proxy configuration
- **dnsmasq** - Internal DNS resolution for custom domains
- **Watchtower** - Automatic container update management

### Applications
- **Portainer** - Docker container management interface
- **Vaultwarden** - Self-hosted password manager
- **Wiki.js** - Modern documentation platform
- **Uptime Kuma** - Service monitoring and status pages

## Network Architecture

The setup creates a secure internal network accessible only through Tailscale:

**DNS Flow:**
1. Client requests `portainer.docker-host.example.com`
2. Tailscale routes DNS query to internal dnsmasq server
3. dnsmasq resolves to Docker host IP
4. Request reaches Nginx Proxy Manager
5. NPM proxies to appropriate service with SSL termination

**Security Benefits:**
- Services never exposed to public internet
- All traffic encrypted by Tailscale
- Valid SSL certificates for professional appearance
- Centralized access control through Tailscale network

## Quick Start

### Prerequisites
- Domain name with Cloudflare DNS management
- Tailscale account and network setup
- Ubuntu Server with Docker installed
- Minimum 4GB RAM, 20GB storage

### Basic Deployment

1. **Clone and configure DNS server:**
```bash
# Deploy internal DNS
mkdir dns-server && cd dns-server
echo "address=/docker-host.example.com/192.168.1.100" > dnsmasq.conf
docker run -d --name internal-dns -p 53:53/udp -v ./dnsmasq.conf:/etc/dnsmasq.conf jpillora/dnsmasq
```

2. **Configure Tailscale SplitDNS:**
   - Add nameserver: `192.168.1.100` restricted to `example.com`
   - Enable DNS override in Tailscale admin

3. **Deploy Nginx Proxy Manager:**
```bash
# Install reverse proxy
docker run -d --name npm -p 80:80 -p 443:443 -p 8181:81 \
  -v npm-data:/data -v npm-ssl:/etc/letsencrypt \
  jc21/nginx-proxy-manager:latest
```

4. **Request wildcard SSL certificate**
5. **Create proxy hosts for each service**

## Advanced Configuration

### Custom Domain Structure

The hierarchical domain structure provides clear organization:

```
service.server.domain.com
├── npm.docker-host.example.com
├── portainer.docker-host.example.com  
├── vault.docker-host.example.com
└── wiki.docker-host.example.com
```

This scales naturally as your infrastructure grows:
- `grafana.monitoring-server.example.com`
- `gitlab.development-server.example.com`
- `nextcloud.storage-server.example.com`

### SSL Certificate Management

Wildcard certificates automatically cover all services under a server subdomain:
- Single certificate for `*.docker-host.example.com`
- Automatic renewal through Let's Encrypt
- DNS challenge validation via Cloudflare API

### Service Discovery

New services integrate seamlessly:
1. Deploy container with exposed port
2. Create NPM proxy host with custom domain
3. Apply existing wildcard certificate
4. Service immediately available via HTTPS

## Security Model

The setup implements defense-in-depth security:

**Network Level:**
- No exposed ports to internet
- Tailscale encryption for all traffic
- Subnet routing through controlled gateway

**Application Level:**
- SSL/TLS termination at proxy
- Individual service authentication
- Container isolation

**Access Control:**
- Device-based access through Tailscale
- User-level service permissions
- Audit logging through monitoring stack

## Operational Benefits

**For Development:**
- Consistent HTTPS in development
- Clean URLs for documentation
- Easy service sharing with team

**For Production:**
- Enterprise-grade security model
- Professional appearance
- Simplified certificate management

**For Learning:**
- Modern networking concepts
- Container orchestration
- Infrastructure as code practices

## Common Use Cases

**Home Office:**
- Internal wiki for documentation
- Password manager for family
- File sharing and synchronization

**Small Business:**
- Customer portal and documentation
- Team collaboration tools
- Secure remote access

**Development Environment:**
- Testing and staging environments
- CI/CD pipeline integration
- Code repository hosting



## Troubleshooting

**DNS Resolution Issues:**
- Verify Tailscale SplitDNS configuration
- Check dnsmasq container logs
- Confirm DNS query routing

**SSL Certificate Problems:**
- Validate Cloudflare API permissions
- Review NPM certificate logs
- Check DNS challenge propagation

**Service Connectivity:**
- Verify container health status
- Check NPM proxy host configuration
- Confirm port availability

## Contributing

Contributions welcome for additional services, security improvements, and documentation updates. Please ensure changes maintain the zero-trust security model and follow the established domain structure.



---


> **Note:** This setup serves as a foundation for advanced networking concepts and can be extended with additional security layers, monitoring solutions, and automation frameworks.
