# Authentication Setup Guide

This guide covers setting up authentication for your secure media server using Authelia and other authentication methods.

## Table of Contents

1. [Authelia Setup](#authelia-setup)
2. [User Management](#user-management)
3. [Two-Factor Authentication](#two-factor-authentication)
4. [LDAP Integration](#ldap-integration)
5. [Basic Authentication Fallback](#basic-authentication-fallback)
6. [Troubleshooting](#troubleshooting)

## Authelia Setup

### Prerequisites

- Docker and Docker Compose installed
- Domain name with DNS control
- SSL certificates configured

### Configuration

1. **Copy configuration files:**
   ```bash
   cp config-examples/authelia/configuration.yml config/authelia/
   cp config-examples/authelia/users_database.yml config/authelia/
   ```

2. **Edit configuration.yml:**
   - Update domain settings
   - Configure session settings
   - Set up notification providers
   - Configure access control rules

3. **Set up users database:**
   - Generate password hashes
   - Define user groups
   - Configure access permissions

### Password Hash Generation

Generate secure password hashes using Authelia:

```bash
docker run --rm authelia/authelia:latest authelia hash-password 'your-secure-password'
```

## User Management

### User Groups

- **admins**: Full access to all services
- **users**: Access to media services and basic functionality
- **family**: Media consumption only
- **guests**: Limited viewing access

### Adding New Users

1. Generate password hash
2. Add user to users_database.yml
3. Assign appropriate groups
4. Restart Authelia container

## Two-Factor Authentication

### Supported Methods

- TOTP (Time-based One-Time Password)
- WebAuthn/FIDO2
- Duo Push notifications

### Setup Process

1. User logs in with username/password
2. Authelia presents 2FA setup options
3. User scans QR code with authenticator app
4. User enters verification code
5. 2FA is enabled for the account

### Recommended Authenticator Apps

- Google Authenticator
- Authy
- Microsoft Authenticator
- 1Password
- Bitwarden

## LDAP Integration

For larger deployments, integrate with existing directory services:

### Active Directory

```yaml
authentication_backend:
  ldap:
    url: ldap://dc.example.com
    base_dn: dc=example,dc=com
    username_attribute: sAMAccountName
    additional_users_dn: ou=users
    users_filter: (&({username_attribute}={input})(objectCategory=person)(objectClass=user))
    additional_groups_dn: ou=groups
    groups_filter: (&(member={dn})(objectClass=group))
    group_name_attribute: cn
    mail_attribute: mail
    display_name_attribute: displayName
    user: cn=authelia,ou=service accounts,dc=example,dc=com
    password: your-service-account-password
```

### OpenLDAP

```yaml
authentication_backend:
  ldap:
    url: ldap://ldap.example.com
    base_dn: dc=example,dc=com
    username_attribute: uid
    additional_users_dn: ou=people
    users_filter: (&({username_attribute}={input})(objectClass=inetOrgPerson))
    additional_groups_dn: ou=groups
    groups_filter: (&(member={dn})(objectClass=groupOfNames))
    group_name_attribute: cn
    mail_attribute: mail
    display_name_attribute: cn
    user: cn=authelia,ou=service,dc=example,dc=com
    password: your-service-account-password
```

## Basic Authentication Fallback

For services that don't support forward auth:

### Nginx Configuration

```nginx
location /admin {
    auth_basic "Admin Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://backend;
}
```

### Generate .htpasswd

```bash
htpasswd -c /etc/nginx/.htpasswd admin
```

## Access Control Rules

### Example Rules

```yaml
access_control:
  default_policy: deny
  rules:
    # Admin access
    - domain: "*.yourdomain.com"
      policy: two_factor
      subject: "group:admins"
    
    # User access to media services
    - domain: "jellyfin.yourdomain.com"
      policy: one_factor
      subject: "group:users"
    
    # Family access to media only
    - domain: "jellyfin.yourdomain.com"
      policy: one_factor
      subject: "group:family"
    
    # Guest access with restrictions
    - domain: "jellyfin.yourdomain.com"
      policy: one_factor
      subject: "group:guests"
      resources:
        - "^/web/.*$"
```

## Session Management

### Configuration

```yaml
session:
  name: authelia_session
  domain: yourdomain.com
  same_site: lax
  secret: your-session-secret
  expiration: 1h
  inactivity: 5m
  remember_me_duration: 1M
```

### Redis Backend

```yaml
session:
  redis:
    host: redis
    port: 6379
    password: your-redis-password
    database_index: 0
```

## Notification Setup

### Email Notifications

```yaml
notifier:
  smtp:
    username: notifications@yourdomain.com
    password: your-email-password
    host: smtp.gmail.com
    port: 587
    sender: "Authelia <notifications@yourdomain.com>"
    subject: "[Authelia] {title}"
    startup_check_address: test@yourdomain.com
```

### File Notifications (Development)

```yaml
notifier:
  filesystem:
    filename: /config/notification.txt
```

## Troubleshooting

### Common Issues

1. **Login loops**
   - Check domain configuration
   - Verify SSL certificates
   - Review access control rules

2. **2FA not working**
   - Verify time synchronization
   - Check authenticator app setup
   - Review TOTP configuration

3. **LDAP connection issues**
   - Test LDAP connectivity
   - Verify service account permissions
   - Check firewall rules

### Debug Mode

Enable debug logging:

```yaml
log:
  level: debug
```

### Log Analysis

Check Authelia logs:

```bash
docker logs authelia
tail -f /var/log/authelia/authelia.log
```

### Testing Configuration

Test configuration without restarting:

```bash
docker exec authelia authelia validate-config /config/configuration.yml
```

## Security Best Practices

1. **Use strong passwords and 2FA**
2. **Regularly rotate secrets**
3. **Monitor authentication logs**
4. **Implement proper access controls**
5. **Keep Authelia updated**
6. **Use secure session settings**
7. **Configure proper CORS settings**
8. **Implement rate limiting**

## Integration Examples

### Traefik Labels

```yaml
labels:
  - "traefik.http.middlewares.authelia.forwardauth.address=http://authelia:9091/api/verify?rd=https://auth.yourdomain.com"
  - "traefik.http.middlewares.authelia.forwardauth.trustForwardHeader=true"
  - "traefik.http.middlewares.authelia.forwardauth.authResponseHeaders=Remote-User,Remote-Groups,Remote-Name,Remote-Email"
```

### Nginx Proxy Manager

Configure forward auth in NPM:
- Forward Hostname/IP: authelia
- Forward Port: 9091
- Forward Auth Path: /api/verify
- Advanced: Add custom headers for user information

## Backup and Recovery

### Backup Configuration

```bash
# Backup Authelia configuration
tar -czf authelia-backup-$(date +%Y%m%d).tar.gz config/authelia/

# Backup user database
cp config/authelia/users_database.yml users_database.yml.backup
```

### Recovery Process

1. Restore configuration files
2. Restart Authelia container
3. Verify user access
4. Test 2FA functionality