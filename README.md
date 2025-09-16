# WebVirtMgr Installation Guide

WebVirtMgr is a free and open-source web-based interface for managing virtual machines through libvirt. It provides a complete KVM management solution with an intuitive web interface, serving as a FOSS alternative to proprietary virtualization management platforms like VMware vSphere, Microsoft System Center Virtual Machine Manager, or Citrix XenCenter.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

### Hardware Requirements
- **CPU**: 64-bit processor with virtualization extensions (Intel VT-x or AMD-V)
- **RAM**: 2GB minimum (4GB+ recommended)
- **Storage**: 20GB minimum for OS and WebVirtMgr
- **Network**: Gigabit Ethernet recommended

### Software Requirements
- **Hypervisor**: KVM/QEMU with libvirt
- **Python**: 2.7 or 3.6+
- **Database**: SQLite (default) or MySQL/PostgreSQL
- **Web Server**: nginx or Apache (nginx recommended)

### Dependencies
- libvirt-python
- Django 1.11+
- python-pip
- supervisor or systemd
- novnc (for VM console access)

### System Access
- root or sudo privileges required
- libvirt group membership for web user

## 2. Supported Operating Systems

This guide supports installation on:
- RHEL 8/9 and derivatives (CentOS Stream, Rocky Linux, AlmaLinux)
- Debian 11/12
- Ubuntu 20.04/22.04/24.04 LTS
- Arch Linux (rolling release)
- Alpine Linux 3.18+
- openSUSE Leap 15.5+ / Tumbleweed
- SUSE Linux Enterprise Server (SLES) 15+
- macOS 12+ (client only - for accessing WebVirtMgr)
- FreeBSD 13+ (experimental)
- Windows (client only - for accessing WebVirtMgr)

## 3. Installation

### RHEL/CentOS/Rocky Linux/AlmaLinux

```bash
# Install EPEL repository
sudo dnf install -y epel-release

# Install required packages
sudo dnf install -y python3 python3-pip git nginx supervisor \
    libvirt-daemon-system libvirt-clients python3-libvirt \
    libxml2-python3 python3-websockify novnc

# Install development tools for Python packages
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y python3-devel libvirt-devel libxml2-devel \
    libxslt-devel mysql-devel

# Clone WebVirtMgr repository
cd /var/www
sudo git clone https://github.com/retspen/webvirtmgr.git
cd webvirtmgr

# Install Python dependencies
sudo pip3 install -r requirements.txt
sudo pip3 install gunicorn

# Configure Django
sudo python3 manage.py syncdb
sudo python3 manage.py collectstatic

# Create default admin user (follow prompts)
sudo python3 manage.py createsuperuser

# Set permissions
sudo chown -R nginx:nginx /var/www/webvirtmgr
```

### Debian/Ubuntu

```bash
# Update package index
sudo apt update

# Install required packages
sudo apt install -y python3 python3-pip git nginx supervisor \
    libvirt-daemon-system libvirt-clients python3-libvirt \
    python3-libxml2 python3-websockify novnc

# Install development packages
sudo apt install -y build-essential python3-dev libvirt-dev \
    libxml2-dev libxslt1-dev zlib1g-dev

# Clone WebVirtMgr repository
cd /var/www
sudo git clone https://github.com/retspen/webvirtmgr.git
cd webvirtmgr

# Install Python dependencies
sudo pip3 install -r requirements.txt
sudo pip3 install gunicorn

# Configure Django
sudo python3 manage.py migrate
sudo python3 manage.py collectstatic

# Create default admin user
sudo python3 manage.py createsuperuser

# Set permissions
sudo chown -R www-data:www-data /var/www/webvirtmgr
```

### Arch Linux

```bash
# Install required packages
sudo pacman -S python python-pip git nginx supervisor \
    libvirt qemu python-libvirt python-lxml novnc

# Install from AUR
yay -S webvirtmgr

# Or manual installation
cd /var/www
sudo git clone https://github.com/retspen/webvirtmgr.git
cd webvirtmgr

# Install Python dependencies
sudo pip install -r requirements.txt
sudo pip install gunicorn

# Configure Django
sudo python manage.py migrate
sudo python manage.py collectstatic
sudo python manage.py createsuperuser

# Set permissions
sudo chown -R http:http /var/www/webvirtmgr
```

### Alpine Linux

```bash
# Install required packages
apk add --no-cache python3 py3-pip git nginx supervisor \
    libvirt-daemon py3-libvirt py3-lxml novnc

# Install build dependencies
apk add --no-cache --virtual .build-deps \
    python3-dev libvirt-dev libxml2-dev libxslt-dev \
    gcc musl-dev linux-headers

# Clone WebVirtMgr
cd /var/www
git clone https://github.com/retspen/webvirtmgr.git
cd webvirtmgr

# Install Python dependencies
pip3 install -r requirements.txt
pip3 install gunicorn

# Configure Django
python3 manage.py migrate
python3 manage.py collectstatic
python3 manage.py createsuperuser

# Clean up build dependencies
apk del .build-deps

# Set permissions
chown -R nginx:nginx /var/www/webvirtmgr
```

### openSUSE/SLES

```bash
# Install required packages
sudo zypper install -y python3 python3-pip git nginx \
    libvirt-daemon python3-libvirt python3-lxml novnc

# Install development packages
sudo zypper install -y python3-devel libvirt-devel \
    libxml2-devel libxslt-devel

# Clone WebVirtMgr
cd /var/www
sudo git clone https://github.com/retspen/webvirtmgr.git
cd webvirtmgr

# Install Python dependencies
sudo pip3 install -r requirements.txt
sudo pip3 install gunicorn

# Configure Django
sudo python3 manage.py migrate
sudo python3 manage.py collectstatic
sudo python3 manage.py createsuperuser

# Set permissions
sudo chown -R wwwrun:www /var/www/webvirtmgr
```

## 4. Configuration

### nginx Configuration

Create `/etc/nginx/conf.d/webvirtmgr.conf`:

```nginx
server {
    listen 80;
    server_name webvirtmgr.example.com;
    
    access_log /var/log/nginx/webvirtmgr_access.log;
    error_log /var/log/nginx/webvirtmgr_error.log;

    location /static/ {
        root /var/www/webvirtmgr;
        expires 30d;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }
}
```

### Gunicorn Configuration

Create `/etc/supervisor/conf.d/webvirtmgr.conf` (Debian/Ubuntu) or `/etc/supervisord.d/webvirtmgr.ini` (RHEL):

```ini
[program:webvirtmgr]
command=/usr/local/bin/gunicorn webvirtmgr.wsgi:application -b 0.0.0.0:8000
directory=/var/www/webvirtmgr
user=nginx
autostart=true
autorestart=true
stdout_logfile=/var/log/webvirtmgr/gunicorn.log
stderr_logfile=/var/log/webvirtmgr/gunicorn_error.log
environment=PATH="/usr/local/bin:/usr/bin"
```

### Console Configuration

Create `/etc/supervisor/conf.d/webvirtmgr-console.conf`:

```ini
[program:webvirtmgr-console]
command=/usr/local/bin/python /var/www/webvirtmgr/console/webvirtmgr-console
directory=/var/www/webvirtmgr
user=nginx
autostart=true
autorestart=true
stdout_logfile=/var/log/webvirtmgr/console.log
stderr_logfile=/var/log/webvirtmgr/console_error.log
```

### libvirt Configuration

Configure libvirt for TCP access:

```bash
# Edit /etc/libvirt/libvirtd.conf
sudo sed -i 's/#listen_tls = 0/listen_tls = 0/g' /etc/libvirt/libvirtd.conf
sudo sed -i 's/#listen_tcp = 1/listen_tcp = 1/g' /etc/libvirt/libvirtd.conf
sudo sed -i 's/#tcp_port = "16509"/tcp_port = "16509"/g' /etc/libvirt/libvirtd.conf
sudo sed -i 's/#auth_tcp = "sasl"/auth_tcp = "none"/g' /etc/libvirt/libvirtd.conf

# For systemd-based systems
sudo systemctl restart libvirtd
```

### Database Configuration (Optional MySQL/PostgreSQL)

For MySQL:
```python
# In settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'webvirtmgr',
        'USER': 'webvirtmgr',
        'PASSWORD': 'your_password',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
```

## 5. Service Management

### systemd (Modern Linux distributions)

Create `/etc/systemd/system/webvirtmgr.service`:

```ini
[Unit]
Description=WebVirtMgr
After=network.target

[Service]
Type=simple
User=nginx
Group=nginx
WorkingDirectory=/var/www/webvirtmgr
Environment="PATH=/usr/local/bin:/usr/bin"
ExecStart=/usr/local/bin/gunicorn webvirtmgr.wsgi:application -b 0.0.0.0:8000
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Create `/etc/systemd/system/webvirtmgr-console.service`:

```ini
[Unit]
Description=WebVirtMgr Console
After=network.target

[Service]
Type=simple
User=nginx
Group=nginx
WorkingDirectory=/var/www/webvirtmgr
ExecStart=/usr/bin/python3 /var/www/webvirtmgr/console/webvirtmgr-console
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Management commands:
```bash
# Enable services
sudo systemctl enable webvirtmgr webvirtmgr-console nginx

# Start services
sudo systemctl start webvirtmgr webvirtmgr-console nginx

# Check status
sudo systemctl status webvirtmgr
```

### Supervisor (Alternative)

```bash
# Start supervisor
sudo systemctl start supervisord

# Reload configuration
sudo supervisorctl reload

# Check status
sudo supervisorctl status

# Start/stop services
sudo supervisorctl start webvirtmgr
sudo supervisorctl stop webvirtmgr
```

## 6. Troubleshooting

### Common Issues

1. **Cannot connect to libvirt**:
```bash
# Check libvirt service
sudo systemctl status libvirtd

# Test connection
virsh -c qemu:///system list

# Add user to libvirt group
sudo usermod -a -G libvirt nginx
```

2. **Console not working**:
```bash
# Check novnc service
sudo systemctl status webvirtmgr-console

# Check WebSocket proxy
ss -tlnp | grep 6080

# Verify novnc installation
which novnc_server
```

3. **Permission denied errors**:
```bash
# Fix ownership
sudo chown -R nginx:nginx /var/www/webvirtmgr

# Fix SELinux context (RHEL/CentOS)
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/webvirtmgr(/.*)?"
sudo restorecon -Rv /var/www/webvirtmgr
```

4. **Database errors**:
```bash
# Reinitialize database
cd /var/www/webvirtmgr
sudo python3 manage.py migrate --run-syncdb
```

### Debug Mode

Enable debug mode in `settings.py`:
```python
DEBUG = True
ALLOWED_HOSTS = ['*']
```

Check logs:
```bash
tail -f /var/log/nginx/webvirtmgr_error.log
tail -f /var/log/webvirtmgr/gunicorn_error.log
```

## 7. Security Considerations

### SSL/TLS Configuration

Add SSL to nginx configuration:

```nginx
server {
    listen 443 ssl http2;
    server_name webvirtmgr.example.com;

    ssl_certificate /etc/letsencrypt/live/webvirtmgr.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/webvirtmgr.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # ... rest of configuration
}

server {
    listen 80;
    server_name webvirtmgr.example.com;
    return 301 https://$server_name$request_uri;
}
```

### Firewall Configuration

```bash
# firewalld (RHEL/CentOS)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=6080/tcp  # VNC console
sudo firewall-cmd --reload

# ufw (Ubuntu/Debian)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 6080/tcp
```

### Authentication Security

1. **Strong passwords**: Enforce strong password policy
2. **Session timeout**: Configure in `settings.py`:
```python
SESSION_COOKIE_AGE = 3600  # 1 hour
SESSION_SAVE_EVERY_REQUEST = True
```

3. **LDAP/AD Integration** (optional):
```python
# Install python-ldap
pip install python-ldap django-auth-ldap

# Configure in settings.py
import ldap
from django_auth_ldap.config import LDAPSearch

AUTHENTICATION_BACKENDS = (
    'django_auth_ldap.backend.LDAPBackend',
    'django.contrib.auth.backends.ModelBackend',
)

AUTH_LDAP_SERVER_URI = "ldap://ldap.example.com"
AUTH_LDAP_BIND_DN = "cn=admin,dc=example,dc=com"
AUTH_LDAP_BIND_PASSWORD = "password"
AUTH_LDAP_USER_SEARCH = LDAPSearch(
    "ou=users,dc=example,dc=com",
    ldap.SCOPE_SUBTREE,
    "(uid=%(user)s)"
)
```

## 8. Performance Tuning

### Gunicorn Optimization

Update gunicorn command for production:
```bash
gunicorn webvirtmgr.wsgi:application \
    -b 0.0.0.0:8000 \
    --workers 4 \
    --timeout 60 \
    --log-level info \
    --access-logfile /var/log/webvirtmgr/access.log
```

### nginx Optimization

```nginx
# In nginx.conf
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 65535;
    use epoll;
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    
    # Compression
    gzip on;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript;
}
```

### Database Optimization

For MySQL backend:
```sql
-- Optimize tables
OPTIMIZE TABLE django_session;
OPTIMIZE TABLE instances_instance;

-- Add indexes
CREATE INDEX idx_instance_name ON instances_instance(name);
CREATE INDEX idx_instance_uuid ON instances_instance(uuid);
```

### System Tuning

```bash
# Increase file descriptors
echo "* soft nofile 65535" >> /etc/security/limits.conf
echo "* hard nofile 65535" >> /etc/security/limits.conf

# Network tuning
echo "net.core.somaxconn = 65535" >> /etc/sysctl.conf
echo "net.ipv4.tcp_max_syn_backlog = 65535" >> /etc/sysctl.conf
sysctl -p
```

## 9. Backup and Restore

### Backup Script

Create `/usr/local/bin/backup-webvirtmgr.sh`:

```bash
#!/bin/bash
BACKUP_DIR="/backup/webvirtmgr"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/webvirtmgr_backup_$DATE.tar.gz"

# Create backup directory
mkdir -p $BACKUP_DIR

# Stop services
systemctl stop webvirtmgr webvirtmgr-console

# Backup database
cd /var/www/webvirtmgr
python3 manage.py dumpdata > $BACKUP_DIR/db_backup_$DATE.json

# Backup files
tar -czf $BACKUP_FILE \
    /var/www/webvirtmgr \
    /etc/nginx/conf.d/webvirtmgr.conf \
    /etc/systemd/system/webvirtmgr*.service \
    $BACKUP_DIR/db_backup_$DATE.json

# Start services
systemctl start webvirtmgr webvirtmgr-console

# Remove old backups (keep 7 days)
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed: $BACKUP_FILE"
```

### Restore Procedure

```bash
#!/bin/bash
BACKUP_FILE="$1"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file>"
    exit 1
fi

# Stop services
systemctl stop webvirtmgr webvirtmgr-console nginx

# Extract backup
tar -xzf $BACKUP_FILE -C /

# Find and restore database
DB_BACKUP=$(tar -tf $BACKUP_FILE | grep "db_backup.*json")
if [ -n "$DB_BACKUP" ]; then
    cd /var/www/webvirtmgr
    python3 manage.py flush --noinput
    python3 manage.py loaddata /$DB_BACKUP
fi

# Fix permissions
chown -R nginx:nginx /var/www/webvirtmgr

# Start services
systemctl start webvirtmgr webvirtmgr-console nginx

echo "Restore completed"
```

## 10. System Requirements

### Minimum Requirements
- **CPU**: 2 cores
- **RAM**: 2GB
- **Storage**: 20GB
- **Network**: 100Mbps

### Recommended Requirements
- **CPU**: 4+ cores
- **RAM**: 4GB+
- **Storage**: 50GB+ SSD
- **Network**: 1Gbps

### Scaling Considerations

For managing 50+ VMs:
- **CPU**: 8+ cores
- **RAM**: 8GB+
- **Database**: External MySQL/PostgreSQL
- **Load Balancer**: HAProxy/nginx for multiple WebVirtMgr instances

## 11. Support

### Official Resources
- **GitHub Repository**: https://github.com/retspen/webvirtmgr
- **Wiki**: https://github.com/retspen/webvirtmgr/wiki
- **Issues**: https://github.com/retspen/webvirtmgr/issues

### Community Support
- **IRC**: #webvirtmgr on Libera.Chat
- **Forums**: Various Linux distribution forums
- **Stack Overflow**: Tag `webvirtmgr`

### Professional Support
- Community-driven project
- Commercial alternatives: oVirt, Proxmox VE

## 12. Contributing

### How to Contribute
1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

### Development Setup
```bash
# Clone repository
git clone https://github.com/yourusername/webvirtmgr.git
cd webvirtmgr

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements-dev.txt

# Run development server
python manage.py runserver 0.0.0.0:8000
```

## 13. License

WebVirtMgr is licensed under the Apache License 2.0. See LICENSE file for details.

Key points:
- Free for commercial use
- Modification allowed
- Distribution allowed
- Patent grant included
- No warranty provided

## 14. Acknowledgments

### Project Credits
- **Anatoliy Guskov**: Original creator and maintainer
- **Contributors**: See GitHub contributors page
- **libvirt Project**: For the virtualization API
- **Django Project**: Web framework
- **noVNC Project**: HTML5 VNC client

### Special Thanks
- KVM/QEMU development team
- Python community
- Open source virtualization community

## 15. Version History

### Current Version
- **Latest Stable**: Check GitHub releases
- **Development**: master branch

### Major Releases
- **v4.8.9**: Latest stable release
- **v4.x**: Django 1.11 support
- **v3.x**: Python 3 support
- **v2.x**: Initial stable releases

### Upgrade Path
```bash
# Backup before upgrade
/usr/local/bin/backup-webvirtmgr.sh

# Pull latest code
cd /var/www/webvirtmgr
git pull origin master

# Update dependencies
pip3 install -r requirements.txt --upgrade

# Run migrations
python3 manage.py migrate

# Restart services
systemctl restart webvirtmgr webvirtmgr-console
```

## 16. Appendices

### A. Port Reference

| Port | Service | Description |
|------|---------|-------------|
| 80/443 | nginx | Web interface |
| 8000 | Gunicorn | Application server |
| 6080 | NoVNC | VNC console proxy |
| 16509 | libvirt | libvirt TCP connection |
| 5900-5999 | VNC | VM console connections |

### B. File Locations

| File/Directory | Purpose |
|---------------|---------|
| `/var/www/webvirtmgr/` | Application root |
| `/etc/nginx/conf.d/webvirtmgr.conf` | nginx configuration |
| `/var/log/webvirtmgr/` | Application logs |
| `/var/lib/webvirtmgr/` | Data directory |

### C. Common Commands

```bash
# Check VM list via CLI
virsh list --all

# Restart all services
systemctl restart webvirtmgr webvirtmgr-console nginx libvirtd

# Django admin shell
cd /var/www/webvirtmgr
python3 manage.py shell

# Create new user
python3 manage.py createsuperuser

# Collect static files after update
python3 manage.py collectstatic --noinput
```

### D. Integration Examples

**API Usage Example**:
```python
import requests

# Login
session = requests.Session()
login_data = {'username': 'admin', 'password': 'password'}
session.post('http://webvirtmgr.example.com/login/', data=login_data)

# Get instance list
response = session.get('http://webvirtmgr.example.com/instances/')
instances = response.json()
```

---

For more detailed information and updates, visit https://github.com/howtomgr/webvirt