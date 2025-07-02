# Secure Media Server Installation Guide

This comprehensive guide will walk you through setting up a secure media server infrastructure with Jellyfin, automated media management, and robust security controls.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [System Preparation](#system-preparation)
3. [Initial Setup](#initial-setup)
4. [Service Configuration](#service-configuration)
5. [Security Configuration](#security-configuration)
6. [SSL/TLS Setup](#ssltls-setup)
7. [Testing and Verification](#testing-and-verification)
8. [Troubleshooting](#troubleshooting)

## Prerequisites

### Hardware Requirements

**Minimum Requirements:**
- CPU: 2 cores (4 cores recommended)
- RAM: 4GB (8GB+ recommended)
- Storage: 100GB system + media storage
- Network: Gigabit Ethernet recommended

**Recommended for 4K Transcoding:**
- CPU: Intel with Quick Sync or dedicated GPU
- RAM: 16GB+
- Storage: NVMe SSD for OS and transcoding cache

### Software Requirements

- **Operating System**: Ubuntu 20.04 LTS or newer
- **Docker**: Version 20.10+
- **Docker Compose**: Version 2.0+
- **Domain Name**: For SSL certificates and external access

### Network Requirements

- **Ports**: 80, 443, 8096 (Jellyfin), 8080 (qBittorrent)
- **Bandwidth**: Sufficient for your streaming needs
- **Static IP**: Recommended for consistent access

## System Preparation

### 1. Update System

```bash
# Update package lists and upgrade system
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y curl wget git unzip htop net-tools ufw
```

### 2. Install Docker

```bash
# Remove old Docker versions
sudo apt remove docker docker-engine docker.io containerd runc

# Install Docker dependencies
sudo apt install -y apt-transport-https ca-certificates gnupg lsb-release

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker
```

### 3. Install Docker Compose

```bash
# Download Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Make executable
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker --version
docker-compose --version
```

## Initial Setup

### 1. Clone Repository

```bash
# Clone the repository
git clone https://github.com/Xieonie/secure-media-server.git
cd secure-media-server

# Make scripts executable
chmod +x scripts/setup/*.sh
chmod +x scripts/maintenance/*.sh
```

### 2. Run Initial Setup Script

```bash
# Run the initial setup script
./scripts/setup/initial-setup.sh

# Follow the prompts to configure:
# - Media storage path
# - Downloads path
# - Configuration path
# - User/Group settings
```

The script will:
- Create necessary users and groups
- Set up directory structure
- Install dependencies
- Configure firewall
- Set up fail2ban
- Create systemd service

### 3. Configure Environment

```bash
# Edit the generated .env file
nano .env

# Key settings to customize:
DOMAIN=your-domain.com
TZ=Your/Timezone
NOTIFICATION_EMAIL=admin@your-domain.com
```

## Service Configuration

### 1. Nginx Proxy Manager Setup

```bash
# Start Nginx Proxy Manager first
cd docker
docker-compose up -d nginx-proxy-manager nginx-db

# Wait for services to start
sleep 30

# Access web interface
echo "Access Nginx Proxy Manager at: http://your-server-ip:81"
echo "Default credentials:"
echo "Email: admin@example.com"
echo "Password: changeme"
```

**Initial Configuration:**
1. Log in and change default password
2. Add SSL certificate for your domain
3. Create proxy hosts for each service

### 2. Authelia Authentication

```bash
# Copy and customize Authelia configuration
cp config-examples/authelia/configuration.yml docker/authelia/
cp config-examples/authelia/users_database.yml docker/authelia/

# Generate secrets
openssl rand -hex 32  # For JWT secret
openssl rand -hex 32  # For session secret
openssl rand -hex 32  # For storage encryption key

# Edit configuration with your secrets
nano docker/authelia/configuration.yml
```

**User Database Setup:**
```bash
# Edit users database
nano docker/authelia/users_database.yml

# Add your users (passwords will be hashed)
# Use: docker run --rm authelia/authelia:latest authelia hash-password 'your-password'
```

### 3. Media Services Configuration

```bash
# Start all services
docker-compose up -d

# Check service status
docker-compose ps

# View logs if needed
docker-compose logs -f [service-name]
```

**Service Access:**
- **Jellyfin**: http://your-server-ip:8096
- **Sonarr**: http://your-server-ip:8989
- **Radarr**: http://your-server-ip:7878
- **Prowlarr**: http://your-server-ip:9696
- **qBittorrent**: http://your-server-ip:8080

## Security Configuration

### 1. Firewall Configuration

```bash
# Configure UFW firewall
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow necessary ports
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Optional: Allow direct access to services (if not using proxy)
sudo ufw allow 8096/tcp  # Jellyfin
sudo ufw allow 8080/tcp  # qBittorrent

# Check status
sudo ufw status verbose
```

### 2. Fail2Ban Configuration

```bash
# Check fail2ban status
sudo systemctl status fail2ban

# View active jails
sudo fail2ban-client status

# Check specific jail
sudo fail2ban-client status sshd
```

### 3. Container Security

**Resource Limits:**
```yaml
# In docker-compose.yml
deploy:
  resources:
    limits:
      memory: 2G
      cpus: '1.0'
    reservations:
      memory: 1G
```

**Security Options:**
```yaml
security_opt:
  - no-new-privileges:true
read_only: true
tmpfs:
  - /tmp
```

## SSL/TLS Setup

### 1. Let's Encrypt Certificates

**Using Nginx Proxy Manager:**
1. Access Nginx Proxy Manager web interface
2. Go to SSL Certificates
3. Add Let's Encrypt certificate
4. Enter your domain and email
5. Enable DNS challenge if needed

**Manual Setup:**
```bash
# Install certbot
sudo apt install certbot python3-certbot-nginx

# Obtain certificate
sudo certbot --nginx -d your-domain.com -d *.your-domain.com

# Test renewal
sudo certbot renew --dry-run
```

### 2. Configure Proxy Hosts

For each service, create a proxy host in Nginx Proxy Manager:

**Jellyfin Proxy Host:**
- Domain: jellyfin.your-domain.com
- Forward Hostname: jellyfin
- Forward Port: 8096
- SSL Certificate: Your Let's Encrypt cert
- Force SSL: Enabled

**Advanced Settings:**
```nginx
# Custom Nginx Configuration for Jellyfin
client_max_body_size 20M;

# Security headers
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

### 3. Authelia Integration

**Forward Auth Configuration:**
```nginx
# In Nginx Proxy Manager Advanced tab
auth_request /authelia;
auth_request_set $target_url $scheme://$http_host$request_uri;
auth_request_set $user $upstream_http_remote_user;
auth_request_set $groups $upstream_http_remote_groups;
auth_request_set $name $upstream_http_remote_name;
auth_request_set $email $upstream_http_remote_email;

proxy_set_header Remote-User $user;
proxy_set_header Remote-Groups $groups;
proxy_set_header Remote-Name $name;
proxy_set_header Remote-Email $email;

error_page 401 =302 https://auth.your-domain.com/?rd=$target_url;

location /authelia {
    internal;
    proxy_pass http://authelia:9091/api/verify;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
    proxy_set_header X-Original-Method $request_method;
    proxy_set_header X-Forwarded-Method $request_method;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $http_host;
    proxy_set_header X-Forwarded-Uri $request_uri;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Authorization $http_authorization;
    proxy_pass_request_headers on;
}
```

## Testing and Verification

### 1. Service Health Checks

```bash
# Check all containers are running
docker-compose ps

# Check container health
docker-compose exec jellyfin curl -f http://localhost:8096/health || echo "Jellyfin health check failed"

# Check logs for errors
docker-compose logs --tail=50 jellyfin
docker-compose logs --tail=50 authelia
```

### 2. Network Connectivity

```bash
# Test internal network connectivity
docker-compose exec jellyfin ping -c 3 authelia
docker-compose exec sonarr ping -c 3 qbittorrent

# Test external access
curl -I https://jellyfin.your-domain.com
curl -I https://auth.your-domain.com
```

### 3. Authentication Testing

1. **Access protected service**: https://jellyfin.your-domain.com
2. **Verify redirect**: Should redirect to Authelia login
3. **Test login**: Use configured credentials
4. **Test 2FA**: If enabled, verify TOTP works
5. **Check access**: Should redirect back to Jellyfin

### 4. SSL Certificate Verification

```bash
# Check certificate details
openssl s_client -connect your-domain.com:443 -servername your-domain.com

# Test SSL rating
curl -s "https://api.ssllabs.com/api/v3/analyze?host=your-domain.com"
```

## Troubleshooting

### Common Issues

#### 1. Container Won't Start

```bash
# Check container logs
docker-compose logs [service-name]

# Check resource usage
docker stats

# Restart specific service
docker-compose restart [service-name]
```

#### 2. Permission Issues

```bash
# Fix ownership
sudo chown -R $PUID:$PGID /path/to/media
sudo chown -R $PUID:$PGID /path/to/config

# Check permissions
ls -la /path/to/media
ls -la /path/to/config
```

#### 3. Network Issues

```bash
# Check Docker networks
docker network ls
docker network inspect secure-media-server_frontend

# Test container connectivity
docker-compose exec jellyfin nslookup authelia
```

#### 4. SSL Certificate Issues

```bash
# Check certificate status
sudo certbot certificates

# Renew certificates
sudo certbot renew

# Check Nginx configuration
sudo nginx -t
```

#### 5. Authentication Issues

```bash
# Check Authelia logs
docker-compose logs authelia

# Verify configuration
docker-compose exec authelia authelia validate-config /config/configuration.yml

# Test Redis connection
docker-compose exec authelia-redis redis-cli ping
```

### Performance Optimization

#### 1. Jellyfin Optimization

```bash
# Enable hardware acceleration (Intel Quick Sync)
# Add to docker-compose.yml:
devices:
  - /dev/dri:/dev/dri

# For NVIDIA GPU:
runtime: nvidia
environment:
  - NVIDIA_VISIBLE_DEVICES=all
```

#### 2. Storage Optimization

```bash
# Use SSD for transcoding cache
# Mount SSD to /config/jellyfin/cache

# Optimize file system
sudo mount -o remount,noatime /path/to/media
```

#### 3. Network Optimization

```bash
# Increase network buffers
echo 'net.core.rmem_max = 134217728' | sudo tee -a /etc/sysctl.conf
echo 'net.core.wmem_max = 134217728' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Monitoring and Maintenance

#### 1. Log Monitoring

```bash
# Monitor logs in real-time
docker-compose logs -f

# Check disk usage
df -h
du -sh /path/to/media/*
```

#### 2. Backup Configuration

```bash
# Run backup script
./scripts/maintenance/backup-configs.sh

# Schedule regular backups
crontab -e
# Add: 0 2 * * * /path/to/secure-media-server/scripts/maintenance/backup-configs.sh
```

#### 3. Update Services

```bash
# Update container images
docker-compose pull
docker-compose up -d

# Clean up old images
docker image prune -f
```

## Next Steps

After successful installation:

1. **Configure Media Libraries**: Set up Jellyfin libraries for movies, TV shows, music
2. **Set Up Indexers**: Configure Prowlarr with your preferred indexers
3. **Configure Download Client**: Set up qBittorrent with VPN if needed
4. **Set Up Monitoring**: Configure Grafana dashboards for system monitoring
5. **Regular Maintenance**: Schedule updates, backups, and health checks

For advanced configurations and troubleshooting, refer to the individual service documentation and the project's GitHub repository.