# Troubleshooting Guide

This guide covers common issues and solutions for the secure media server infrastructure.

## Table of Contents

1. [General Troubleshooting](#general-troubleshooting)
2. [Docker Issues](#docker-issues)
3. [Network Problems](#network-problems)
4. [Authentication Issues](#authentication-issues)
5. [Media Server Problems](#media-server-problems)
6. [VPN Issues](#vpn-issues)
7. [Performance Problems](#performance-problems)
8. [Security Issues](#security-issues)

## General Troubleshooting

### Basic Diagnostic Steps

1. **Check System Resources**
   ```bash
   # Check disk space
   df -h
   
   # Check memory usage
   free -h
   
   # Check CPU usage
   top
   
   # Check system load
   uptime
   ```

2. **Check Service Status**
   ```bash
   # Check Docker service
   systemctl status docker
   
   # Check container status
   docker ps -a
   
   # Check container logs
   docker logs <container_name>
   ```

3. **Check Network Connectivity**
   ```bash
   # Test internet connectivity
   ping 8.8.8.8
   
   # Test DNS resolution
   nslookup google.com
   
   # Check open ports
   netstat -tlnp
   ```

### Log Analysis

#### Centralized Logging
```bash
# View all container logs
docker-compose logs

# Follow logs in real-time
docker-compose logs -f

# View specific service logs
docker-compose logs jellyfin

# View logs with timestamps
docker-compose logs -t --since="1h"
```

#### System Logs
```bash
# Check system journal
journalctl -f

# Check specific service
journalctl -u docker

# Check kernel messages
dmesg | tail -20
```

## Docker Issues

### Container Won't Start

#### Symptoms
- Container exits immediately
- "Exited (1)" status
- Port binding errors

#### Diagnosis
```bash
# Check container logs
docker logs <container_name>

# Check Docker daemon logs
journalctl -u docker

# Inspect container configuration
docker inspect <container_name>

# Check resource usage
docker stats
```

#### Solutions

1. **Port Conflicts**
   ```bash
   # Check what's using the port
   netstat -tlnp | grep :8080
   
   # Kill process using port
   sudo kill -9 <PID>
   
   # Change port in docker-compose.yml
   ports:
     - "8081:8080"  # Use different external port
   ```

2. **Permission Issues**
   ```bash
   # Fix ownership of config directories
   sudo chown -R 1000:1000 ./config/
   
   # Set correct permissions
   chmod -R 755 ./config/
   ```

3. **Resource Constraints**
   ```bash
   # Check available resources
   docker system df
   
   # Clean up unused resources
   docker system prune -a
   
   # Increase memory limits
   deploy:
     resources:
       limits:
         memory: 2G
   ```

### Container Performance Issues

#### High CPU Usage
```bash
# Monitor container resources
docker stats

# Check container processes
docker exec <container> top

# Limit CPU usage
deploy:
  resources:
    limits:
      cpus: '1.0'
```

#### High Memory Usage
```bash
# Check memory usage
docker exec <container> free -h

# Monitor memory over time
watch docker stats

# Set memory limits
deploy:
  resources:
    limits:
      memory: 1G
```

### Docker Compose Issues

#### Service Dependencies
```yaml
# Ensure proper startup order
services:
  database:
    # ... database config
    
  app:
    depends_on:
      database:
        condition: service_healthy
    # ... app config
```

#### Network Issues
```bash
# List Docker networks
docker network ls

# Inspect network configuration
docker network inspect <network_name>

# Test connectivity between containers
docker exec <container1> ping <container2>
```

## Network Problems

### Cannot Access Services

#### External Access Issues
1. **Check Firewall Rules**
   ```bash
   # Check UFW status
   sudo ufw status
   
   # Allow specific ports
   sudo ufw allow 8080/tcp
   
   # Check iptables rules
   sudo iptables -L -n
   ```

2. **Check Reverse Proxy Configuration**
   ```bash
   # Test Nginx Proxy Manager
   curl -I http://localhost:81
   
   # Check Traefik dashboard
   curl -I http://localhost:8080
   
   # Verify SSL certificates
   openssl s_client -connect yourdomain.com:443
   ```

3. **DNS Resolution Issues**
   ```bash
   # Check DNS configuration
   cat /etc/resolv.conf
   
   # Test DNS resolution
   nslookup yourdomain.com
   
   # Check hosts file
   cat /etc/hosts
   ```

#### Internal Network Issues
```bash
# Check Docker network connectivity
docker exec jellyfin ping sonarr

# Verify service discovery
docker exec jellyfin nslookup sonarr

# Check network configuration
docker network inspect media-network
```

### SSL/TLS Issues

#### Certificate Problems
```bash
# Check certificate validity
openssl x509 -in cert.pem -text -noout

# Test SSL connection
openssl s_client -connect yourdomain.com:443

# Check certificate chain
curl -I https://yourdomain.com
```

#### Let's Encrypt Issues
```bash
# Check Certbot logs
sudo journalctl -u certbot

# Manual certificate renewal
sudo certbot renew --dry-run

# Check certificate expiration
sudo certbot certificates
```

## Authentication Issues

### Authelia Problems

#### Login Failures
1. **Check Authelia Logs**
   ```bash
   docker logs authelia
   
   # Look for authentication errors
   docker logs authelia | grep -i "authentication failed"
   ```

2. **Verify User Configuration**
   ```bash
   # Check users database
   cat config/authelia/users_database.yml
   
   # Verify password hash
   docker run --rm authelia/authelia:latest authelia hash-password 'password'
   ```

3. **LDAP Integration Issues**
   ```bash
   # Test LDAP connectivity
   ldapsearch -x -H ldap://ldap.server.com -D "cn=user,dc=domain,dc=com" -W
   
   # Check LDAP configuration
   cat config/authelia/configuration.yml | grep -A 20 ldap
   ```

#### 2FA Issues
```bash
# Check TOTP configuration
cat config/authelia/configuration.yml | grep -A 10 totp

# Verify time synchronization
timedatectl status

# Reset user 2FA
# Remove user from Authelia database and re-register
```

### Session Management
```bash
# Check Redis connectivity
docker exec redis redis-cli ping

# View active sessions
docker exec redis redis-cli keys "authelia:*"

# Clear user sessions
docker exec redis redis-cli flushdb
```

## Media Server Problems

### Jellyfin Issues

#### Transcoding Problems
1. **Hardware Acceleration**
   ```bash
   # Check GPU availability
   lspci | grep -i vga
   
   # Verify GPU drivers
   nvidia-smi  # For NVIDIA
   vainfo      # For Intel/AMD
   
   # Check device permissions
   ls -la /dev/dri/
   ```

2. **FFmpeg Issues**
   ```bash
   # Check FFmpeg installation
   docker exec jellyfin ffmpeg -version
   
   # Test transcoding manually
   docker exec jellyfin ffmpeg -i /media/test.mkv -c:v libx264 -f null -
   ```

#### Library Scanning Issues
```bash
# Check file permissions
ls -la /media/

# Verify mount points
mount | grep media

# Check Jellyfin logs
docker logs jellyfin | grep -i "scan"
```

### Sonarr/Radarr Issues

#### Download Client Connection
```bash
# Test qBittorrent API
curl -u admin:password http://localhost:8080/api/v2/app/version

# Check download client configuration
cat config/sonarr/config.xml | grep -A 5 "DownloadClient"

# Verify network connectivity
docker exec sonarr ping qbittorrent
```

#### Indexer Problems
```bash
# Test indexer connectivity
curl -I https://indexer.example.com

# Check Prowlarr logs
docker logs prowlarr | grep -i "indexer"

# Verify API keys
cat config/sonarr/config.xml | grep -i "apikey"
```

### qBittorrent Issues

#### WebUI Access Problems
```bash
# Check qBittorrent logs
docker logs qbittorrent

# Verify WebUI settings
cat config/qbittorrent/qBittorrent/config/qBittorrent.conf | grep WebUI

# Test local access
curl -I http://localhost:8080
```

#### Download Issues
```bash
# Check disk space
df -h /downloads

# Verify permissions
ls -la /downloads/

# Check torrent status
# Access WebUI and check torrent list
```

## VPN Issues

### Connection Problems

#### VPN Won't Connect
```bash
# Check VPN container logs
docker logs vpn

# Test VPN configuration
docker exec vpn ping 8.8.8.8

# Verify VPN credentials
cat config/vpn/auth.txt
```

#### Kill Switch Issues
```bash
# Check iptables rules
docker exec vpn iptables -L -n

# Test kill switch
docker stop vpn
docker exec qbittorrent ping 8.8.8.8  # Should fail
```

### IP Leaks

#### DNS Leak Detection
```bash
# Check DNS servers
docker exec vpn cat /etc/resolv.conf

# Test DNS leak
docker exec vpn nslookup google.com

# Use online tools
curl -s https://www.dnsleaktest.com/
```

#### IP Leak Detection
```bash
# Check external IP
docker exec vpn curl -s https://ipinfo.io/ip

# Compare with VPN provider's IP ranges
# Should match VPN server location
```

## Performance Problems

### Slow Streaming

#### Network Bandwidth
```bash
# Test network speed
speedtest-cli

# Check network utilization
iftop

# Monitor container network usage
docker stats --format "table {{.Container}}\t{{.NetIO}}"
```

#### Transcoding Performance
```bash
# Monitor CPU usage during transcoding
top -p $(pgrep ffmpeg)

# Check GPU utilization
nvidia-smi -l 1  # For NVIDIA
intel_gpu_top    # For Intel

# Optimize transcoding settings in Jellyfin
# Lower quality settings or enable hardware acceleration
```

### High Resource Usage

#### CPU Optimization
```bash
# Identify high CPU processes
top -o %CPU

# Set CPU limits for containers
deploy:
  resources:
    limits:
      cpus: '2.0'

# Optimize container settings
# Reduce concurrent operations
```

#### Memory Optimization
```bash
# Check memory usage
free -h

# Monitor container memory
docker stats --format "table {{.Container}}\t{{.MemUsage}}"

# Set memory limits
deploy:
  resources:
    limits:
      memory: 2G
```

#### Storage Performance
```bash
# Check disk I/O
iotop

# Test disk speed
hdparm -t /dev/sda

# Optimize storage configuration
# Use SSD for transcoding cache
# Separate media and config storage
```

## Security Issues

### Failed Login Attempts

#### Fail2ban Configuration
```bash
# Check Fail2ban status
sudo fail2ban-client status

# View banned IPs
sudo fail2ban-client status authelia

# Unban IP address
sudo fail2ban-client set authelia unbanip 192.168.1.100
```

#### Log Analysis
```bash
# Check authentication logs
grep "authentication failed" /var/log/authelia/authelia.log

# Monitor failed attempts
tail -f /var/log/auth.log | grep "Failed"

# Check Nginx access logs
tail -f /var/log/nginx/access.log | grep "401\|403"
```

### Security Scanning

#### Vulnerability Assessment
```bash
# Scan for open ports
nmap -sS localhost

# Check for outdated packages
apt list --upgradable

# Scan Docker images
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image jellyfin/jellyfin:latest
```

#### Container Security
```bash
# Check container privileges
docker inspect <container> | grep -i privileged

# Verify user permissions
docker exec <container> id

# Check mounted volumes
docker inspect <container> | grep -A 10 "Mounts"
```

## Emergency Procedures

### Service Recovery

#### Complete System Recovery
```bash
# Stop all services
docker-compose down

# Check system resources
df -h && free -h

# Restart Docker service
sudo systemctl restart docker

# Start services one by one
docker-compose up -d database
docker-compose up -d authelia
docker-compose up -d jellyfin
# ... continue with other services
```

#### Database Recovery
```bash
# Backup current database
docker exec postgres pg_dump -U authelia authelia > backup.sql

# Restore from backup
docker exec -i postgres psql -U authelia authelia < backup.sql

# Check database integrity
docker exec postgres psql -U authelia -c "SELECT version();"
```

### Data Recovery

#### Configuration Backup
```bash
# Create configuration backup
tar -czf config-backup-$(date +%Y%m%d).tar.gz config/

# Restore configuration
tar -xzf config-backup-20240101.tar.gz
```

#### Media Library Recovery
```bash
# Check filesystem integrity
sudo fsck /dev/sdb1

# Mount recovery disk
sudo mount /dev/sdb1 /mnt/recovery

# Copy media files
rsync -av /mnt/recovery/media/ /media/
```

## Monitoring and Alerting

### Health Checks

#### Automated Monitoring
```bash
#!/bin/bash
# health-check.sh

services=("jellyfin" "sonarr" "radarr" "qbittorrent" "authelia")

for service in "${services[@]}"; do
    if docker ps | grep -q "$service"; then
        echo "$service: Running"
    else
        echo "$service: Stopped"
        # Send alert
        echo "$service is down" | mail -s "Service Alert" admin@domain.com
    fi
done
```

#### Performance Monitoring
```bash
# Monitor resource usage
watch docker stats

# Check service response times
curl -w "@curl-format.txt" -o /dev/null -s http://localhost:8096/health

# Monitor disk space
df -h | awk '$5 > 80 {print $0}' | mail -s "Disk Space Alert" admin@domain.com
```

### Log Rotation

#### Configure Log Rotation
```bash
# /etc/logrotate.d/docker-containers
/var/lib/docker/containers/*/*.log {
    rotate 7
    daily
    compress
    size=1M
    missingok
    delaycompress
    copytruncate
}
```

## Getting Help

### Community Resources
- **Jellyfin**: https://forum.jellyfin.org/
- **Sonarr**: https://forums.sonarr.tv/
- **Radarr**: https://radarr.video/
- **Authelia**: https://github.com/authelia/authelia/discussions
- **Docker**: https://forums.docker.com/

### Documentation
- **Jellyfin Docs**: https://jellyfin.org/docs/
- **Authelia Docs**: https://www.authelia.com/docs/
- **Docker Docs**: https://docs.docker.com/
- **Traefik Docs**: https://doc.traefik.io/traefik/

### Professional Support
- Consider professional support for production environments
- Regular security audits and penetration testing
- Backup and disaster recovery planning
- Performance optimization consulting