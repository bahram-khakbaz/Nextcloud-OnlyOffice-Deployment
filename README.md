```markdown
# Nextcloud + OnlyOffice Deployment Guide
## Production-Ready Docker Setup with Persian (Farsi) Font Support and RTL Notes

This document provides a complete, step-by-step guide to deploying Nextcloud with OnlyOffice Document Server using Docker. It includes Persian font installation, RTL behavior observations, LDAP authentication setup, security hardening, and troubleshooting tips.

Tested configuration:
- Nextcloud 32.0.3
- OnlyOffice Document Server latest (9.2.1 at time of testing)
- PostgreSQL 16
- Docker + Docker Compose
- Security Hardening
---

## 1. Operating System Preparation

### 1.1 Update System Packages
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl ca-certificates gnupg lsb-release unzip unrar
```

### 1.2 (Optional) Set System Locale
```bash
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8
```

---

## 2. Docker & Docker Compose Installation

### 2.1 Install Docker Engine
```bash
curl -fsSL https://get.docker.com | sh
sudo systemctl enable docker
sudo systemctl start docker
```

### 2.2 Install Docker Compose Plugin
```bash
sudo apt install -y docker-compose-plugin
```

Verify installation:
```bash
docker version
docker compose version
```

---

## 3. Directory Structure
```bash
sudo mkdir -p /data/nextcloud/docker/{db,nextcloud_data,nextcloud_config}
sudo mkdir -p /opt/nextcloud-docker/onlyoffice-fonts
sudo mkdir -p /opt/nextcloud-docker/php
sudo mkdir -p /opt/nextcloud-docker/apache
```

---

## 4. Docker Compose Configuration
Create the file `/opt/nextcloud-docker/docker-compose.yml`:

```yaml
version: '3.8'
networks:
  nextcloud-net:
    driver: bridge

services:
  db:
    image: postgres:16
    container_name: nextcloud-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: nextcloud
      POSTGRES_USER: ncuser
      POSTGRES_PASSWORD: 'CHANGE_TO_SECURE_PASSWORD'
    volumes:
      - /data/nextcloud/docker/db:/var/lib/postgresql/data
    networks:
      - nextcloud-net

  app:
    image: nextcloud:32
    container_name: nextcloud-app
    depends_on:
      - db
    ports:
      - "80:80"
    environment:
      POSTGRES_DB: nextcloud
      POSTGRES_USER: ncuser
      POSTGRES_PASSWORD: 'CHANGE_TO_SECURE_PASSWORD'
      POSTGRES_HOST: db
    volumes:
      - /data/nextcloud/docker/nextcloud_data:/var/www/html/data
      - /data/nextcloud/docker/nextcloud_config:/var/www/html/config
      - /opt/nextcloud-docker/php/custom.ini:/usr/local/etc/php/conf.d/custom.ini:ro
      - /opt/nextcloud-docker/apache/security.conf:/etc/apache2/conf-available/security.conf:ro
    restart: unless-stopped
    networks:
      - nextcloud-net

  onlyoffice:
    image: onlyoffice/documentserver:latest
    container_name: onlyoffice
    restart: unless-stopped
    # ports:  # External port removed for security - use internal communication only
    #   - "8081:80"
    environment:
      JWT_ENABLED: 'true'
      JWT_SECRET: 'CHANGE_TO_SECURE_JWT_SECRET'
      JWT_HEADER: 'Authorization'
    volumes:
      - /opt/nextcloud-docker/onlyoffice-fonts:/usr/share/fonts/truetype/custom
    networks:
      - nextcloud-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/healthcheck"]
      interval: 30s
      timeout: 5s
      retries: 5
```

Start the stack:
```bash
cd /opt/nextcloud-docker
docker compose up -d
```

---

## 5. Nextcloud Initial Setup
1. Open browser: `http://SERVER_IP`
2. Create an admin account
3. Database configuration:
   - Database type: PostgreSQL
   - Username: `ncuser`
   - Password: (the one set in compose)
   - Database name: `nextcloud`
   - Host: `db`

---

## 6. Install and Configure OnlyOffice Integration
Install the app inside Nextcloud:
```bash
docker exec -u www-data nextcloud-app php occ app:install onlyoffice
docker exec -u www-data nextcloud-app php occ app:enable onlyoffice
```

Configure in **Nextcloud → Settings → Administration → ONLYOFFICE** (internal-only recommended for security):
- Document Editing Service address: `http://onlyoffice`
- Advanced server settings:
  - Document server address for internal requests: `http://onlyoffice`
  - Server address for internal requests from the Document Editing Service: `http://app`
- Enable JWT and set secret matching the one in `docker-compose.yml`
- Header: `Authorization`

Save and test the connection.

---

## 7. LDAP Authentication Integration (Active Directory or OpenLDAP)

### 7.1 Install and Enable LDAP App
```bash
docker exec -u www-data nextcloud-app php occ app:install user_ldap
docker exec -u www-data nextcloud-app php occ app:enable user_ldap
```

### 7.2 Activate LDAP Provider Factory
Add to `/data/nextcloud/docker/nextcloud_config/config.php`:
```php
'ldapProviderFactory' => 'OCA\\User_LDAP\\LDAPProviderFactory',
```

### 7.3 Configure LDAP in Nextcloud GUI
Go to **Nextcloud → Settings → Administration → LDAP / AD integration** and follow standard configuration for your LDAP/AD server.

---

## 8. Persian Font Installation for OnlyOffice

### 8.1 Download and Place Fonts on Host
Example using Vazir (recommended modern Persian font):
```bash
cd /opt/nextcloud-docker/onlyoffice-fonts
wget https://github.com/rastikerdar/vazir-font/releases/download/v27.2.2/Vazir-Regular.ttf
wget https://github.com/rastikerdar/vazir-font/releases/download/v27.2.2/Vazir-Bold.ttf
# Add more weights if desired
```

(Alternative: Sahel, B Nazanin, or any TTF Persian font collection.)

### 8.2 Refresh Font Cache and Generate AllFonts.js
```bash
docker exec -it onlyoffice fc-cache -fv
docker exec -it onlyoffice /usr/bin/documentserver-generate-allfonts.sh
docker compose restart onlyoffice
```

Verify fonts are detected:
```bash
docker exec -it onlyoffice fc-list | grep -i vazir
docker exec -it onlyoffice grep -i vazir /var/www/onlyoffice/documentserver/server/FileConverter/bin/AllFonts.js
```

---

## 9. Persian RTL Behavior Notes

### 9.1 Word Documents (DOCX)
- Full RTL support
- Correct text shaping and ligatures
- Persian fonts selectable and render properly

### 9.2 Spreadsheets (XLSX) – Known Limitation
- Text may appear disconnected or reversed
- RTL alignment is inconsistent
- Root cause: OnlyOffice Spreadsheet Editor has limited RTL shaping support for Persian/Arabic
- This is an upstream OnlyOffice limitation (not a configuration issue)
- Workaround: Use Collabora Online for better spreadsheet RTL if needed

### 9.3 General Recommendations
- Set Nextcloud language to Persian (fa) for full UI RTL
- Use modern fonts like Vazir or Sahel for best results

---

## 10. Security Hardening

### 10.1 Hide PHP and Apache Version Headers
Create `/opt/nextcloud-docker/php/custom.ini`:
```
expose_php = Off
```

Create `/opt/nextcloud-docker/apache/security.conf`:
```
ServerTokens Prod
ServerSignature Off
Header always unset X-Powered-By
```

These files are mounted in the `app` service volumes (see compose file).

Apply changes:
```bash
docker compose down
docker compose up -d
```

### 10.2 Block /status.php (Prevent Version Leak)
Rename the file (Nextcloud does not require it):
```bash
docker exec -it nextcloud-app mv /var/www/html/status.php /var/www/html/status.php.disabled
docker compose restart app
```

### 10.3 Close OnlyOffice External Port
Remove the ports section from the onlyoffice service in compose (use internal communication only).

Apply changes:
```bash
docker compose down
docker compose up -d
```

---

## 11. Key config.php Settings (Reference)
Important sections (passwords/JWT masked):
```php
'trusted_domains' => 
array (
  0 => 'YOUR_DOMAIN_OR_IP',
  1 => 'YOUR_SERVER_IP',
  2 => 'onlyoffice',
  3 => 'nextcloud-app',
  4 => '127.0.0.1',
  5 => 'app',
),
'overwrite.cli.url' => 'http://YOUR_SERVER_IP',
'overwriteprotocol' => 'http',
'onlyoffice' => 
array (
  'jwt_enabled' => 'true',
  'jwt_secret' => 'YOUR_JWT_SECRET',
  'jwt_header' => 'Authorization',
  'internal_url' => 'http://onlyoffice/',
  'public_url' => 'http://YOUR_SERVER_IP:8081/',
),
'ldapProviderFactory' => 'OCA\\User_LDAP\\LDAPProviderFactory',
```

---

## 12. Troubleshooting

### 12.1 OnlyOffice Connection Issues
- Verify JWT secret match
- Use internal URL `http://onlyoffice` for server-to-server requests
- Check trusted_domains includes container names

### 12.2 Font Issues
- Re-run `fc-cache` and `documentserver-generate-allfonts.sh`
- Restart OnlyOffice container

### 12.3 LDAP Issues
- Check LDAP server connectivity from container
- Verify bind DN and password

### 12.4 General Validation
```bash
docker ps                    # All containers healthy
docker logs onlyoffice | grep Version   # Confirm version
docker exec onlyoffice curl http://localhost/healthcheck   # Should return OK
```

---

## 13. Production Recommendations
- Use reverse proxy (Nginx/Traefik) with HTTPS and proper domain
- Regular backups of volumes
- Monitor resource usage (OnlyOffice is RAM-intensive)
- Update images periodically and re-run font generation

This setup is stable, secure, and optimized for Persian language use with LDAP authentication and security hardening.
```
😊
