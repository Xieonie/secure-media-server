# Media Server Network Security Guide

## Overview

This document outlines the network security architecture and best practices for the secure media server infrastructure. The focus is on protecting media content, preventing unauthorized access, and maintaining service availability while ensuring optimal performance.

## Network Architecture

### Segmentation Strategy

```
Internet
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                    Reverse Proxy                            │
│              (Nginx Proxy Manager)                         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │   SSL/TLS   │ │   WAF       │ │   Rate Limiting     │   │
│  │ Termination │ │ Protection  │ │                     │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                    Docker Network                          │
│                 (media-server-network)                     │
│                                                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │   Jellyfin  │ │   Sonarr    │ │     Radarr          │   │
│  │   :8096     │ │   :8989     │ │     :7878           │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
│                                                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │  Prowlarr   │ │ qBittorrent │ │    Monitoring       │   │
│  │   :9696     │ │   :8080     │ │  (Grafana/Prom)     │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                    Storage Layer                           │
│                                                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │   Media     │ │  Downloads  │ │   Configuration     │   │
│  │  Storage    │ │   Storage   │ │     Storage         │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Network Zones

#### DMZ Zone (Reverse Proxy)
- **Purpose**: Public-facing services and SSL termination
- **Components**: Nginx Proxy Manager, SSL certificates
- **Security Level**: High
- **Access**: Internet → DMZ only

#### Application Zone (Docker Network)
- **Purpose**: Media server applications
- **Components**: Jellyfin, Sonarr, Radarr, Prowlarr, qBittorrent
- **Security Level**: Medium-High
- **Access**: DMZ → Application, Internal → Application

#### Storage Zone
- **Purpose**: Media and configuration storage
- **Components**: NFS/SMB shares, local storage
- **Security Level**: High
- **Access**: Application → Storage only

## Docker Network Security

### Custom Bridge Network

```yaml
networks:
  media-server:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1
    driver_opts:
      com.docker.network.bridge.name: media-br0
      com.docker.network.bridge.enable_icc: "true"
      com.docker.network.bridge.enable_ip_masquerade: "true"
```

### Container Isolation

#### Network Policies
```yaml
# Example service with network restrictions
jellyfin:
  networks:
    media-server:
      ipv4_address: 172.20.0.10
  # Restrict container capabilities
  cap_drop:
    - ALL
  cap_add:
    - CHOWN
    - SETGID
    - SETUID
  # Security options
  security_opt:
    - no-new-privileges:true
  # Read-only root filesystem
  read_only: true
  tmpfs:
    - /tmp
    - /var/tmp
```

#### Inter-Container Communication

**Allowed Communications:**
- Reverse Proxy → All application containers
- Sonarr/Radarr → Prowlarr (indexer management)
- Sonarr/Radarr → qBittorrent (download management)
- All containers → Monitoring (metrics collection)

**Blocked Communications:**
- qBittorrent → Sonarr/Radarr (prevent reverse connections)
- External → qBittorrent (except through VPN)
- Containers → Host system (except mounted volumes)

## Firewall Configuration

### Host-Level Firewall (UFW)

```bash
# Default policies
ufw default deny incoming
ufw default allow outgoing

# SSH access (change port from default)
ufw allow 2222/tcp

# HTTP/HTTPS for reverse proxy
ufw allow 80/tcp
ufw allow 443/tcp

# Docker daemon (if remote access needed)
# ufw allow from 192.168.1.0/24 to any port 2376

# Monitoring (restrict to management network)
ufw allow from 192.168.1.0/24 to any port 3000  # Grafana
ufw allow from 192.168.1.0/24 to any port 9090  # Prometheus

# Enable firewall
ufw enable
```

### Docker Firewall Rules

```bash
# Create custom chain for media server
iptables -N MEDIA-SERVER-FILTER

# Default deny for Docker networks
iptables -I DOCKER-USER -i docker0 -j MEDIA-SERVER-FILTER
iptables -A MEDIA-SERVER-FILTER -j DROP

# Allow established connections
iptables -I MEDIA-SERVER-FILTER -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# Allow internal Docker communication
iptables -I MEDIA-SERVER-FILTER -s 172.20.0.0/16 -d 172.20.0.0/16 -j ACCEPT

# Allow reverse proxy to applications
iptables -I MEDIA-SERVER-FILTER -s 172.20.0.2 -d 172.20.0.0/16 -j ACCEPT

# Block direct external access to applications
iptables -A MEDIA-SERVER-FILTER -d 172.20.0.0/16 -j DROP
```

## SSL/TLS Configuration

### Certificate Management

#### Let's Encrypt Integration
```yaml
# Nginx Proxy Manager SSL configuration
ssl_certificate /etc/letsencrypt/live/media.yourdomain.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/media.yourdomain.com/privkey.pem;

# SSL protocols and ciphers
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA384;
ssl_prefer_server_ciphers off;

# HSTS
add_header Strict-Transport-Security "max-age=63072000" always;
```

#### Certificate Monitoring
```bash
# Check certificate expiration
openssl x509 -in /etc/letsencrypt/live/media.yourdomain.com/cert.pem -noout -dates

# Automated renewal check
certbot renew --dry-run
```

### Internal TLS

#### Container-to-Container Encryption
```yaml
# Example: Secure communication between services
environment:
  - JELLYFIN_HTTPS_PORT=8920
  - JELLYFIN_CERT_PATH=/config/ssl/jellyfin.crt
  - JELLYFIN_KEY_PATH=/config/ssl/jellyfin.key
```

## Access Control

### Authentication Layers

#### 1. Reverse Proxy Authentication
```nginx
# Basic authentication for admin interfaces
location /admin {
    auth_basic "Admin Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
    
    proxy_pass http://backend;
    proxy_set_header X-Remote-User $remote_user;
}
```

#### 2. Application-Level Authentication
```yaml
# Jellyfin authentication configuration
authentication:
  providers:
    - name: "Local"
      type: "local"
      enabled: true
    - name: "LDAP"
      type: "ldap"
      enabled: false
      server: "ldap://ldap.yourdomain.com"
```

#### 3. Network-Level Access Control
```yaml
# IP-based access restrictions
access_control:
  admin_networks:
    - "192.168.1.0/24"    # Management network
    - "10.0.0.0/8"        # VPN network
  
  user_networks:
    - "192.168.1.0/24"    # Local network
    - "10.0.0.0/8"        # VPN network
    - "0.0.0.0/0"         # Internet (with authentication)
```

### Role-Based Access Control

#### User Roles
```yaml
roles:
  admin:
    permissions:
      - manage_users
      - manage_libraries
      - manage_settings
      - view_logs
      - manage_plugins
    
  power_user:
    permissions:
      - manage_own_library
      - download_content
      - transcode_content
    
  user:
    permissions:
      - view_content
      - create_playlists
      - rate_content
    
  guest:
    permissions:
      - view_public_content
    restrictions:
      - no_download
      - limited_bandwidth
```

## VPN Integration

### Torrent Traffic Isolation

#### VPN Container Configuration
```yaml
vpn:
  image: dperson/openvpn-client
  cap_add:
    - NET_ADMIN
  devices:
    - /dev/net/tun
  environment:
    - VPN_AUTH=username;password
  volumes:
    - ./config/vpn:/vpn:ro
  sysctls:
    - net.ipv6.conf.all.disable_ipv6=1
```

#### qBittorrent VPN Binding
```yaml
qbittorrent:
  network_mode: "service:vpn"
  depends_on:
    - vpn
  # Remove port mappings - use VPN container's network
```

### Kill Switch Implementation
```bash
# Ensure traffic only goes through VPN
iptables -A OUTPUT -o tun0 -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -d 192.168.1.0/24 -j ACCEPT
iptables -A OUTPUT -j DROP
```

## Monitoring and Logging

### Network Traffic Monitoring

#### Prometheus Network Metrics
```yaml
# Node exporter network metrics
node_exporter:
  command:
    - '--collector.netdev'
    - '--collector.netstat'
    - '--collector.conntrack'
```

#### Custom Network Monitoring
```bash
# Monitor container network usage
docker stats --format "table {{.Container}}\t{{.NetIO}}" --no-stream

# Monitor connection states
ss -tuln | grep -E ':(8096|8989|7878|9696|8080)'
```

### Security Event Logging

#### Nginx Access Logs
```nginx
# Custom log format for security analysis
log_format security '$remote_addr - $remote_user [$time_local] '
                   '"$request" $status $bytes_sent '
                   '"$http_referer" "$http_user_agent" '
                   '"$http_x_forwarded_for" "$request_time"';

access_log /var/log/nginx/security.log security;
```

#### Application Security Logs
```yaml
# Jellyfin security logging
logging:
  level: Information
  file:
    path: /config/logs/jellyfin.log
    max_file_size: 100MB
    retained_file_count: 10
  
  # Enable security-related logging
  categories:
    - "Jellyfin.Api.Auth"
    - "Jellyfin.Api.Controllers"
    - "Microsoft.AspNetCore.Authentication"
```

### Intrusion Detection

#### Fail2Ban Configuration
```ini
# /etc/fail2ban/jail.local
[nginx-limit-req]
enabled = true
filter = nginx-limit-req
action = iptables-multiport[name=ReqLimit, port="http,https", protocol=tcp]
logpath = /var/log/nginx/error.log
findtime = 600
bantime = 7200
maxretry = 10

[jellyfin-auth]
enabled = true
filter = jellyfin-auth
action = iptables-allports[name=jellyfin, protocol=all]
logpath = /opt/jellyfin/config/logs/jellyfin.log
findtime = 600
bantime = 3600
maxretry = 5
```

## Backup and Recovery

### Network Configuration Backup

#### Automated Backup Script
```bash
#!/bin/bash
# Network configuration backup

BACKUP_DIR="/backup/network-config"
DATE=$(date +%Y%m%d_%H%M%S)

# Backup firewall rules
iptables-save > "$BACKUP_DIR/iptables_$DATE.rules"
ufw status numbered > "$BACKUP_DIR/ufw_$DATE.status"

# Backup Docker network configuration
docker network ls --format "{{.Name}}" | while read network; do
    docker network inspect "$network" > "$BACKUP_DIR/docker_network_${network}_$DATE.json"
done

# Backup SSL certificates
tar -czf "$BACKUP_DIR/ssl_certs_$DATE.tar.gz" /etc/letsencrypt/

# Backup Nginx configuration
tar -czf "$BACKUP_DIR/nginx_config_$DATE.tar.gz" /etc/nginx/
```

### Disaster Recovery

#### Network Recovery Procedures
1. **Restore Docker Networks**
   ```bash
   # Recreate custom networks
   docker network create --driver bridge media-server
   ```

2. **Restore Firewall Rules**
   ```bash
   # Restore iptables
   iptables-restore < /backup/network-config/iptables_latest.rules
   
   # Restore UFW
   ufw --force reset
   # Reapply UFW rules from backup
   ```

3. **Restore SSL Certificates**
   ```bash
   # Extract certificates
   tar -xzf /backup/network-config/ssl_certs_latest.tar.gz -C /
   
   # Restart nginx
   systemctl restart nginx
   ```

## Security Hardening

### Container Security

#### Security Scanning
```bash
# Scan container images for vulnerabilities
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy image jellyfin/jellyfin:latest

# Scan for secrets in images
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    trufflesecurity/trufflehog:latest docker --image jellyfin/jellyfin:latest
```

#### Runtime Security
```yaml
# Security-hardened container configuration
security_opt:
  - no-new-privileges:true
  - seccomp:unconfined  # Only if required
  - apparmor:docker-default

# Resource limits
deploy:
  resources:
    limits:
      cpus: '2.0'
      memory: 4G
    reservations:
      cpus: '0.5'
      memory: 1G

# Health checks
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8096/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

### Network Hardening

#### Disable Unnecessary Services
```bash
# Disable unused network services
systemctl disable avahi-daemon
systemctl disable cups
systemctl disable bluetooth

# Remove unused network protocols
echo 'install dccp /bin/true' >> /etc/modprobe.d/blacklist-rare-network.conf
echo 'install sctp /bin/true' >> /etc/modprobe.d/blacklist-rare-network.conf
echo 'install rds /bin/true' >> /etc/modprobe.d/blacklist-rare-network.conf
```

#### Kernel Network Security
```bash
# /etc/sysctl.d/99-network-security.conf
# IP forwarding (only if needed)
net.ipv4.ip_forward = 0

# Ignore ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Ignore source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# Log martian packets
net.ipv4.conf.all.log_martians = 1

# Ignore ping requests
net.ipv4.icmp_echo_ignore_all = 1

# TCP SYN flood protection
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 3
```

## Compliance and Auditing

### Security Auditing

#### Regular Security Checks
```bash
#!/bin/bash
# Security audit script

echo "=== Network Security Audit ==="
echo "Date: $(date)"
echo

# Check open ports
echo "Open ports:"
ss -tuln

# Check firewall status
echo -e "\nFirewall status:"
ufw status verbose

# Check for suspicious connections
echo -e "\nActive connections:"
netstat -an | grep ESTABLISHED

# Check SSL certificate expiration
echo -e "\nSSL certificate status:"
for cert in /etc/letsencrypt/live/*/cert.pem; do
    domain=$(basename $(dirname "$cert"))
    expiry=$(openssl x509 -in "$cert" -noout -enddate | cut -d= -f2)
    echo "$domain: $expiry"
done

# Check container security
echo -e "\nContainer security status:"
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### Compliance Reporting

#### Automated Compliance Checks
```bash
# Check compliance with security standards
# - CIS Docker Benchmark
# - NIST Cybersecurity Framework
# - OWASP Container Security

docker run --rm -it --net host --pid host --userns host --cap-add audit_control \
    -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
    -v /etc:/etc:ro \
    -v /usr/bin/containerd:/usr/bin/containerd:ro \
    -v /usr/bin/runc:/usr/bin/runc:ro \
    -v /usr/lib/systemd:/usr/lib/systemd:ro \
    -v /var/lib:/var/lib:ro \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    --label docker_bench_security \
    docker/docker-bench-security
```

## Incident Response

### Network Incident Response

#### Isolation Procedures
```bash
# Isolate compromised container
docker network disconnect media-server compromised_container

# Block suspicious IP
iptables -I INPUT -s SUSPICIOUS_IP -j DROP

# Enable enhanced logging
iptables -I INPUT -s SUSPICIOUS_IP -j LOG --log-prefix "BLOCKED: "
```

#### Forensic Data Collection
```bash
# Capture network traffic
tcpdump -i any -w /tmp/incident_$(date +%s).pcap host SUSPICIOUS_IP

# Collect container logs
docker logs compromised_container > /tmp/container_logs_$(date +%s).log

# Export container filesystem
docker export compromised_container > /tmp/container_export_$(date +%s).tar
```

### Recovery Procedures

#### Service Recovery
1. **Stop affected services**
2. **Restore from clean backup**
3. **Update security configurations**
4. **Restart services with monitoring**
5. **Verify security posture**

#### Network Recovery
1. **Reset firewall rules**
2. **Recreate Docker networks**
3. **Restore SSL certificates**
4. **Verify connectivity**
5. **Resume monitoring**

## Best Practices Summary

### Network Security Checklist

- [ ] Implement network segmentation
- [ ] Configure proper firewall rules
- [ ] Use SSL/TLS for all communications
- [ ] Implement strong authentication
- [ ] Monitor network traffic
- [ ] Regular security updates
- [ ] Backup network configurations
- [ ] Test disaster recovery procedures
- [ ] Conduct regular security audits
- [ ] Maintain incident response procedures

### Ongoing Maintenance

- **Daily**: Monitor logs and alerts
- **Weekly**: Review access logs and user activity
- **Monthly**: Update security configurations
- **Quarterly**: Conduct security assessments
- **Annually**: Review and update security policies