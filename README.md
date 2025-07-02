# Secure Media Server Infrastructure 🎬🔒

This repository documents the implementation of a hardened media server using Docker Compose, with a focus on security through network segmentation, access controls, and minimal attack surface. The setup includes popular media management applications like Plex/Jellyfin, Sonarr, Radarr, and related tools, all deployed with security best practices.

**Important Note:** This setup is designed for personal/home use. Ensure you comply with all applicable laws and licensing requirements for media content in your jurisdiction.

## 🎯 Goals

* Deploy a comprehensive media management and streaming solution
* Minimize attack surface through containerization and network isolation
* Implement proper access controls and authentication
* Ensure data protection and backup strategies
* Provide secure remote access capabilities
* Monitor system health and security events

## 🛠️ Technologies Used

* [Docker](https://www.docker.com/) & [Docker Compose](https://docs.docker.com/compose/)
* [Jellyfin](https://jellyfin.org/) or [Plex](https://www.plex.tv/) - Media server
* [Sonarr](https://sonarr.tv/) - TV series management
* [Radarr](https://radarr.video/) - Movie management
* [Prowlarr](https://prowlarr.com/) - Indexer manager
* [qBittorrent](https://www.qbittorrent.org/) - BitTorrent client
* [Nginx Proxy Manager](https://nginxproxymanager.com/) - Reverse proxy with SSL
* [Traefik](https://traefik.io/) - Alternative reverse proxy (optional)
* [Authelia](https://www.authelia.com/) - Authentication and authorization server
* [Fail2Ban](https://www.fail2ban.org/) - Intrusion prevention
* [Watchtower](https://containrrr.dev/watchtower/) - Automated container updates

## ✨ Key Features/Highlights

* **Containerized Architecture:** All services run in isolated Docker containers
* **Network Segmentation:** Custom Docker networks for service isolation
* **Reverse Proxy:** SSL termination and subdomain routing
* **Authentication:** Centralized authentication with Authelia
* **VPN Integration:** Optional VPN container for torrent traffic
* **Automated Updates:** Container image updates with Watchtower
* **Health Monitoring:** Service health checks and alerting
* **Backup Integration:** Automated backup of configuration and metadata
* **Resource Limits:** CPU and memory constraints for each service
* **Security Headers:** Proper HTTP security headers configuration

## 🏛️ Repository Structure

```
secure-media-server/
├── README.md
├── docker/
│   ├── docker-compose.yml
│   ├── docker-compose.override.yml.example
│   └── .env.example
├── config-examples/
│   ├── nginx-proxy-manager/
│   │   └── nginx-custom.conf
│   ├── authelia/
│   │   ├── configuration.yml
│   │   └── users_database.yml
│   ├── traefik/
│   │   ├── traefik.yml
│   │   └── dynamic.yml
│   └── fail2ban/
│       ├── jail.local
│       └── filter.d/
├── scripts/
│   ├── setup/
│   │   ├── initial-setup.sh
│   │   └── ssl-setup.sh
│   ├── maintenance/
│   │   ├── backup-configs.sh
│   │   ├── update-containers.sh
│   │   └── health-check.sh
│   └── security/
│       ├── security-audit.sh
│       └── log-analysis.sh
└── docs/
    ├── installation-guide.md
    ├── network-security.md
    ├── authentication-setup.md
    ├── vpn-configuration.md
    └── troubleshooting.md
```

## 🚀 Getting Started / Configuration

### Prerequisites

1. **System Requirements:**
   - Linux server (Ubuntu 20.04+ recommended)
   - Docker and Docker Compose installed
   - Minimum 4GB RAM, 8GB+ recommended
   - Sufficient storage for media content

2. **Network Setup:**
   - Static IP address for the server
   - Domain name with DNS control (for SSL certificates)
   - Router port forwarding (if remote access needed)

### Quick Start

1. **Clone the repository:**
   ```bash
   git clone https://github.com/Xieonie/secure-media-server.git
   cd secure-media-server
   ```

2. **Configure environment:**
   ```bash
   cp docker/.env.example docker/.env
   # Edit .env file with your specific settings
   ```

3. **Set up directory structure:**
   ```bash
   ./scripts/setup/initial-setup.sh
   ```

4. **Start the media server stack:**
   ```bash
   cd docker
   docker-compose up -d
   ```

5. **Configure SSL certificates:**
   ```bash
   ./scripts/setup/ssl-setup.sh
   ```

## 🔧 Service Configuration

### Core Media Services

#### Jellyfin Media Server
- **Purpose:** Stream media content to devices
- **Security:** Authentication required, HTTPS only
- **Network:** Isolated media network
- **Storage:** Read-only access to media, read-write to config

#### Sonarr & Radarr
- **Purpose:** Automated TV show and movie management
- **Security:** Behind authentication proxy
- **API Keys:** Secured with strong random keys
- **Network:** Isolated from internet-facing services

#### qBittorrent
- **Purpose:** BitTorrent client for content acquisition
- **Security:** VPN-only traffic, web UI authentication
- **Network:** Isolated download network
- **Storage:** Dedicated download directory

### Security Services

#### Nginx Proxy Manager
- **Purpose:** Reverse proxy with SSL termination
- **Features:** Automatic Let's Encrypt certificates
- **Security:** Rate limiting, security headers
- **Configuration:** Custom nginx configurations for hardening

#### Authelia
- **Purpose:** Authentication and authorization
- **Features:** 2FA support, session management
- **Security:** LDAP/file-based user database
- **Integration:** Protects all web interfaces

#### Fail2Ban
- **Purpose:** Intrusion prevention system
- **Monitoring:** Web server logs, authentication failures
- **Actions:** IP blocking, email notifications
- **Configuration:** Custom filters for media server logs

## 🌐 Network Architecture

### Docker Networks

```yaml
networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
  
  media:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 172.21.0.0/24
  
  downloads:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 172.22.0.0/24
```

### Network Segmentation

1. **Frontend Network:** Reverse proxy and authentication services
2. **Media Network:** Media server and management applications
3. **Downloads Network:** BitTorrent client and VPN services
4. **Internal Networks:** No direct internet access for sensitive services

## 🔐 Security Implementation

### Access Control

1. **Authentication Layers:**
   - Authelia for centralized authentication
   - Individual service authentication as backup
   - Strong password policies enforced

2. **Network Security:**
   - Docker network isolation
   - Firewall rules (UFW/iptables)
   - VPN-only access for sensitive operations

3. **SSL/TLS:**
   - Let's Encrypt certificates for all services
   - HSTS headers enforced
   - TLS 1.2+ only

### Container Security

1. **Resource Limits:**
   ```yaml
   deploy:
     resources:
       limits:
         memory: 1G
         cpus: '0.5'
   ```

2. **User Permissions:**
   - Non-root users in containers
   - Proper file permissions (PUID/PGID)
   - Read-only filesystems where possible

3. **Security Scanning:**
   - Regular vulnerability scans
   - Automated security updates
   - Image signature verification

### Data Protection

1. **Backup Strategy:**
   - Configuration backups to remote storage
   - Database backups for metadata
   - Media content protection strategies

2. **Encryption:**
   - Encrypted storage for sensitive data
   - Encrypted communication channels
   - Secure key management

## 📊 Monitoring and Maintenance

### Health Monitoring

1. **Container Health:**
   - Docker health checks
   - Resource usage monitoring
   - Service availability checks

2. **Security Monitoring:**
   - Failed authentication attempts
   - Unusual network traffic
   - File integrity monitoring

3. **Performance Monitoring:**
   - Media streaming quality
   - Download speeds and ratios
   - Storage space utilization

### Automated Maintenance

1. **Updates:**
   - Watchtower for container updates
   - System package updates
   - Security patch management

2. **Cleanup:**
   - Log rotation and cleanup
   - Temporary file removal
   - Old backup cleanup

## 🔧 Advanced Configuration

### VPN Integration

For enhanced privacy, especially for torrent traffic:

```yaml
vpn:
  image: dperson/openvpn-client
  cap_add:
    - NET_ADMIN
  devices:
    - /dev/net/tun
  volumes:
    - ./config/vpn:/vpn:ro
  environment:
    - VPN_CONFIG=/vpn/config.ovpn
```

### Custom Authentication

Authelia configuration for advanced authentication:

```yaml
authentication_backend:
  file:
    path: /config/users_database.yml
    password:
      algorithm: argon2id
      iterations: 1
      salt_length: 16
      parallelism: 8
      memory: 64
```

## 🔮 Potential Improvements/Future Plans

* Integration with external authentication providers (LDAP, OAuth)
* Advanced monitoring with Prometheus and Grafana
* Automated content organization with machine learning
* Integration with cloud storage providers
* Mobile app for remote management
* Advanced network security with IDS/IPS
* Compliance monitoring and reporting

## ⚠️ Security Considerations

* **Legal Compliance:** Ensure all content acquisition complies with local laws
* **Network Exposure:** Minimize internet-facing services
* **Regular Updates:** Keep all components updated with security patches
* **Access Logging:** Monitor and log all access attempts
* **Backup Security:** Encrypt and secure all backup data
* **VPN Usage:** Consider VPN for all torrent traffic

## 📚 Additional Resources

* [Docker Security Best Practices](https://docs.docker.com/engine/security/)
* [Jellyfin Security Documentation](https://jellyfin.org/docs/general/administration/security/)
* [Authelia Documentation](https://www.authelia.com/docs/)
* [Nginx Security Headers](https://securityheaders.com/)
* [Media Server Security Guide](https://github.com/Cloudbox/Community/wiki/Security)