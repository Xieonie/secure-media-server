# VPN Configuration Guide

This guide covers setting up VPN integration for your secure media server to protect torrent traffic and enhance privacy.

## Table of Contents

1. [VPN Overview](#vpn-overview)
2. [VPN Provider Selection](#vpn-provider-selection)
3. [OpenVPN Configuration](#openvpn-configuration)
4. [WireGuard Configuration](#wireguard-configuration)
5. [Container Integration](#container-integration)
6. [Kill Switch Setup](#kill-switch-setup)
7. [DNS Configuration](#dns-configuration)
8. [Troubleshooting](#troubleshooting)

## VPN Overview

### Why Use a VPN for Media Server?

1. **Privacy Protection**: Hide your IP address from trackers and peers
2. **ISP Throttling**: Bypass ISP throttling of torrent traffic
3. **Geographic Restrictions**: Access content from different regions
4. **Legal Protection**: Additional layer of privacy for legal content
5. **Network Security**: Encrypt traffic between server and internet

### VPN Integration Architecture

```
Internet
    ↓
VPN Provider
    ↓
VPN Container (OpenVPN/WireGuard)
    ↓
Torrent Client Container (qBittorrent)
    ↓
Media Management (Sonarr/Radarr)
    ↓
Local Network
```

## VPN Provider Selection

### Recommended VPN Providers

#### Tier 1 (Excellent for Media Servers)
- **Mullvad**: Privacy-focused, supports WireGuard, port forwarding
- **ProtonVPN**: Strong privacy, good speeds, port forwarding
- **IVPN**: No-logs policy, WireGuard support, privacy-focused

#### Tier 2 (Good Options)
- **NordVPN**: Large server network, good speeds
- **ExpressVPN**: Fast speeds, good reliability
- **Surfshark**: Unlimited connections, good value

### Selection Criteria

```yaml
vpn_requirements:
  privacy:
    - no_logs_policy: "verified"
    - jurisdiction: "privacy_friendly"
    - payment_methods: "anonymous_options"
    
  technical:
    - protocols: ["OpenVPN", "WireGuard"]
    - port_forwarding: "supported"
    - kill_switch: "built_in"
    - dns_leak_protection: "included"
    
  performance:
    - server_locations: "multiple_countries"
    - bandwidth: "unlimited"
    - connection_speed: "high"
    - uptime: ">99%"
    
  compatibility:
    - linux_support: "native"
    - docker_support: "official_images"
    - api_access: "available"
```

## OpenVPN Configuration

### Docker Compose Integration

```yaml
version: '3.8'

services:
  vpn:
    image: dperson/openvpn-client:latest
    container_name: vpn
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun
    volumes:
      - ./config/vpn:/vpn:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      - VPN_CONFIG=/vpn/config.ovpn
      - VPN_AUTH=/vpn/auth.txt
      - FIREWALL=on
      - ROUTE_1=192.168.0.0/16
    networks:
      - vpn-network
    dns:
      - 1.1.1.1
      - 1.0.0.1
    sysctls:
      - net.ipv6.conf.all.disable_ipv6=1

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    network_mode: "service:vpn"
    depends_on:
      - vpn
    volumes:
      - ./config/qbittorrent:/config
      - /mnt/downloads:/downloads
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - WEBUI_PORT=8080

networks:
  vpn-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### OpenVPN Configuration File

```bash
# /config/vpn/config.ovpn
client
dev tun
proto udp
remote vpn-server.provider.com 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
remote-cert-tls server
cipher AES-256-CBC
auth SHA256
comp-lzo
verb 3
mute 20

# Kill switch - block traffic if VPN disconnects
script-security 2
up /vpn/up.sh
down /vpn/down.sh

# DNS settings
dhcp-option DNS 1.1.1.1
dhcp-option DNS 1.0.0.1
```

### Authentication File

```bash
# /config/vpn/auth.txt
your-vpn-username
your-vpn-password
```

### VPN Scripts

#### Up Script (/config/vpn/up.sh)
```bash
#!/bin/bash
# Script runs when VPN connects

echo "VPN connected: $(date)" >> /var/log/vpn.log

# Allow local network access
iptables -I OUTPUT -d 192.168.0.0/16 -j ACCEPT
iptables -I OUTPUT -d 172.16.0.0/12 -j ACCEPT
iptables -I OUTPUT -d 10.0.0.0/8 -j ACCEPT

# Block all other traffic except through VPN
iptables -A OUTPUT -o tun+ -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -j DROP
```

#### Down Script (/config/vpn/down.sh)
```bash
#!/bin/bash
# Script runs when VPN disconnects

echo "VPN disconnected: $(date)" >> /var/log/vpn.log

# Remove firewall rules
iptables -F OUTPUT
```

## WireGuard Configuration

### WireGuard Docker Setup

```yaml
version: '3.8'

services:
  wireguard:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: wireguard
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - ./config/wireguard:/config
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    networks:
      - vpn-network

  qbittorrent-wg:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent-wg
    restart: unless-stopped
    network_mode: "service:wireguard"
    depends_on:
      - wireguard
    volumes:
      - ./config/qbittorrent:/config
      - /mnt/downloads:/downloads
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - WEBUI_PORT=8080
```

### WireGuard Configuration

```ini
# /config/wireguard/wg0.conf
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = 10.2.0.2/32
DNS = 1.1.1.1, 1.0.0.1

# Kill switch
PostUp = iptables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PreDown = iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT

[Peer]
PublicKey = VPN_PROVIDER_PUBLIC_KEY
Endpoint = vpn-server.provider.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

## Container Integration

### Network Isolation

```yaml
# Isolated VPN network
networks:
  vpn-network:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1

  media-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.21.0.0/16
          gateway: 172.21.0.1
```

### Service Dependencies

```yaml
services:
  # VPN must start first
  vpn:
    # ... VPN configuration
    healthcheck:
      test: ["CMD", "ping", "-c", "1", "1.1.1.1"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Torrent client depends on VPN
  qbittorrent:
    depends_on:
      vpn:
        condition: service_healthy
    # ... other configuration

  # Media managers connect to torrent client
  sonarr:
    depends_on:
      - qbittorrent
    networks:
      - media-network
    # ... other configuration
```

## Kill Switch Setup

### iptables Kill Switch

```bash
#!/bin/bash
# /config/vpn/killswitch.sh

# Flush existing rules
iptables -F
iptables -X

# Default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow local network
iptables -A INPUT -s 192.168.0.0/16 -j ACCEPT
iptables -A OUTPUT -d 192.168.0.0/16 -j ACCEPT
iptables -A INPUT -s 172.16.0.0/12 -j ACCEPT
iptables -A OUTPUT -d 172.16.0.0/12 -j ACCEPT
iptables -A INPUT -s 10.0.0.0/8 -j ACCEPT
iptables -A OUTPUT -d 10.0.0.0/8 -j ACCEPT

# Allow VPN connection
iptables -A OUTPUT -p udp --dport 1194 -j ACCEPT
iptables -A INPUT -p udp --sport 1194 -j ACCEPT

# Allow traffic through VPN tunnel
iptables -A INPUT -i tun+ -j ACCEPT
iptables -A OUTPUT -o tun+ -j ACCEPT

# Allow DNS through VPN
iptables -A OUTPUT -o tun+ -p udp --dport 53 -j ACCEPT
iptables -A INPUT -i tun+ -p udp --sport 53 -j ACCEPT

# Block everything else
iptables -A OUTPUT -j DROP
iptables -A INPUT -j DROP
```

### Systemd Kill Switch Service

```ini
# /etc/systemd/system/vpn-killswitch.service
[Unit]
Description=VPN Kill Switch
After=network.target

[Service]
Type=oneshot
ExecStart=/config/vpn/killswitch.sh
RemainAfterExit=yes
ExecStop=/sbin/iptables -F

[Install]
WantedBy=multi-user.target
```

## DNS Configuration

### VPN DNS Settings

```yaml
# Docker Compose DNS configuration
services:
  vpn:
    dns:
      - 1.1.1.1      # Cloudflare
      - 1.0.0.1      # Cloudflare
      - 9.9.9.9      # Quad9
      - 149.112.112.112  # Quad9
    environment:
      - DNS_SERVERS=1.1.1.1,1.0.0.1
```

### DNS Leak Prevention

```bash
#!/bin/bash
# Check for DNS leaks

echo "Checking DNS servers..."
nslookup google.com

echo "Checking external IP..."
curl -s https://ipinfo.io/ip

echo "Checking DNS leak..."
curl -s https://www.dnsleaktest.com/
```

### Custom DNS Configuration

```bash
# /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1 1.0.0.1
FallbackDNS=9.9.9.9 149.112.112.112
Domains=~.
DNSSEC=yes
DNSOverTLS=yes
Cache=yes
```

## Monitoring and Health Checks

### VPN Health Check Script

```bash
#!/bin/bash
# /config/vpn/health-check.sh

LOG_FILE="/var/log/vpn-health.log"

log() {
    echo "$(date): $1" >> "$LOG_FILE"
}

# Check VPN connection
if ping -c 1 1.1.1.1 &>/dev/null; then
    log "VPN connection: OK"
else
    log "VPN connection: FAILED"
    # Restart VPN container
    docker restart vpn
fi

# Check IP address
EXTERNAL_IP=$(curl -s https://ipinfo.io/ip)
EXPECTED_REGION="VPN_PROVIDER_REGION"

if [[ "$EXTERNAL_IP" =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    log "External IP: $EXTERNAL_IP"
else
    log "Failed to get external IP"
fi

# Check torrent client
if curl -s http://localhost:8080 &>/dev/null; then
    log "qBittorrent: OK"
else
    log "qBittorrent: FAILED"
fi
```

### Monitoring Dashboard

```yaml
# Prometheus monitoring
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./config/prometheus:/etc/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-storage:/var/lib/grafana
    ports:
      - "3000:3000"
```

## Troubleshooting

### Common Issues

#### 1. VPN Won't Connect
```bash
# Check VPN logs
docker logs vpn

# Test VPN configuration
openvpn --config /config/vpn/config.ovpn --verb 5

# Check firewall rules
iptables -L -n
```

#### 2. DNS Leaks
```bash
# Check DNS configuration
cat /etc/resolv.conf

# Test DNS resolution
nslookup google.com 1.1.1.1

# Check for leaks
curl -s https://www.dnsleaktest.com/
```

#### 3. No Internet Access
```bash
# Check routing table
ip route show

# Test connectivity
ping 8.8.8.8
ping google.com

# Check iptables rules
iptables -L -n -v
```

#### 4. Torrent Client Not Working
```bash
# Check container networking
docker exec qbittorrent ip addr show

# Test port forwarding
curl -s https://portchecker.co/check?port=PORT_NUMBER

# Check qBittorrent logs
docker logs qbittorrent
```

### Performance Optimization

#### 1. VPN Server Selection
```bash
# Test VPN server speeds
for server in server1 server2 server3; do
    echo "Testing $server..."
    ping -c 5 $server.provider.com
done
```

#### 2. Protocol Optimization
```bash
# OpenVPN optimization
comp-lzo yes
fast-io
sndbuf 524288
rcvbuf 524288

# WireGuard optimization
MTU = 1420
```

#### 3. Container Resource Limits
```yaml
services:
  vpn:
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.5'
        reservations:
          memory: 128M
          cpus: '0.25'
```

## Security Best Practices

### 1. VPN Provider Security
- Choose providers with verified no-logs policies
- Use providers in privacy-friendly jurisdictions
- Enable kill switch functionality
- Use strong authentication methods

### 2. Container Security
- Run containers as non-root users
- Use read-only filesystems where possible
- Limit container capabilities
- Regular security updates

### 3. Network Security
- Isolate VPN traffic from other services
- Use strong firewall rules
- Monitor for DNS leaks
- Regular security audits

### 4. Operational Security
- Regular VPN server rotation
- Monitor connection logs
- Use secure DNS providers
- Implement proper backup strategies