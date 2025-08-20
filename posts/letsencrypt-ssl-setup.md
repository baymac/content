---
title: 'Lets Encrypt SSL Setup with Nginx'
date: '2025-08-20'
tags: letsencrypt, nginx, dns
ai-gen: true
---

This guide shows how to obtain SSL certificates using Nginx, then remove Nginx while keeping the certificates and auto-renewal for use with Docker containers.

## Prerequisites

- Ubuntu 24.04 LTS (or similar Debian-based system)
- Root access to the server
- Domain pointing to your server's IP address
- Ports 80 and 443 accessible from the internet

## Step 1: System Update and Package Installation

Update your system and install required packages:

```bash
# Update package list
apt update

# Install Certbot, Nginx, and the Certbot Nginx plugin
apt install -y certbot python3-certbot-nginx nginx
```

## Step 2: Verify Nginx is Running

Check that Nginx started successfully:

```bash
systemctl status nginx
```

You should see `Active: active (running)` in the output.

## Step 3: Create Basic Nginx Configuration

Create a basic configuration for your domain (replace `your-domain.com` with your actual domain):

```bash
cat > /etc/nginx/sites-available/your-domain.com << 'NGINXEOF'
server {
    listen 80;
    server_name your-domain.com;

    location / {
        return 200 'Hello from your-domain.com - SSL setup in progress';
        add_header Content-Type text/plain;
    }

    # For Let's Encrypt verification
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }
}
NGINXEOF
```

Enable the site:

```bash
# Create symbolic link to enable the site
ln -s /etc/nginx/sites-available/your-domain.com /etc/nginx/sites-enabled/

# Test Nginx configuration
nginx -t

# Reload Nginx
systemctl reload nginx
```

## Step 4: Verify DNS Configuration

Make sure your domain points to your server:

```bash
# Check your server's public IP
curl -4 ifconfig.me

# Check if domain resolves to your IP
nslookup your-domain.com

# Alternative: Check with Google DNS
nslookup your-domain.com 8.8.8.8
```

Test HTTP access:

```bash
# Test with domain (if DNS has propagated)
curl http://your-domain.com

# Alternative: Test with IP and Host header
curl -H "Host: your-domain.com" http://YOUR_SERVER_IP
```

## Step 5: Obtain SSL Certificate

Use Certbot to obtain and install the SSL certificate:

```bash
certbot --nginx -d your-domain.com --agree-tos --no-eff-email --email your-email@domain.com
```

**Important notes:**
- Replace `your-domain.com` with your actual domain
- Replace `your-email@domain.com` with your actual email
- The `--agree-tos` flag accepts Let's Encrypt's Terms of Service
- The `--no-eff-email` flag opts out of EFF communications

## Step 6: Verify SSL Certificate Installation

Check that the certificate was installed successfully:

```bash
# List all certificates
certbot certificates

# View the updated Nginx configuration
cat /etc/nginx/sites-available/your-domain.com
```

You should see SSL configuration has been automatically added by Certbot.

Test HTTPS access:

```bash
# Test HTTPS (if DNS has propagated)
curl https://your-domain.com

# Alternative: Test with IP and Host header
curl -H "Host: your-domain.com" https://YOUR_SERVER_IP --insecure
```

## Step 7: Verify Auto-Renewal Setup

Check that automatic renewal is configured:

```bash
# Check Certbot timer status
systemctl status certbot.timer

# Test renewal process (dry run)
certbot renew --dry-run
```

## Step 8: Remove Nginx (Keep Certificates)

Now that we have the certificates, we can remove Nginx while preserving the certificates and renewal system:

### 8.1 Stop and Disable Nginx

```bash
# Stop Nginx service
systemctl stop nginx

# Disable Nginx from starting at boot
systemctl disable nginx
```

### 8.2 Remove Nginx Packages

```bash
# Remove Nginx and related packages (keep Certbot)
apt remove --purge -y nginx nginx-common python3-certbot-nginx

# Clean up Nginx configuration directory
rm -rf /etc/nginx
```

### 8.3 Re-enable Certbot Auto-Renewal

Since removing the Nginx plugin may have disabled the timer, re-enable it:

```bash
# Enable and start Certbot timer
systemctl enable certbot.timer
systemctl start certbot.timer

# Verify timer is active
systemctl status certbot.timer
```

## Step 9: Verify Final Setup

### 9.1 Check Certificate Location

```bash
# List certificates
certbot certificates

# Check certificate files
ls -la /etc/letsencrypt/live/your-domain.com/
```

### 9.2 Verify Ports are Free

```bash
# Check that ports 80 and 443 are free for Docker
ss -tlnp | grep -E ':80|:443'
```

No output means the ports are free.

### 9.3 Test Certificate Renewal

```bash
# Test renewal (this won't actually renew unless close to expiry)
certbot renew --dry-run
```

## Step 10: Using Certificates with Docker

Now you can use the certificates in your Docker containers:

### Docker Compose Example

```yaml
version: '3.8'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro  # Mount certificates
    restart: unless-stopped
```

### Nginx Configuration for Docker

```nginx
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # Include SSL security settings
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        # Proxy to your application
        proxy_pass http://your-app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Certificate Information

- **Certificate Path:** `/etc/letsencrypt/live/your-domain.com/fullchain.pem`
- **Private Key Path:** `/etc/letsencrypt/live/your-domain.com/privkey.pem`
- **Certificate Type:** ECDSA (modern and secure)
- **Validity:** 90 days (auto-renewed)
- **Auto-Renewal:** Runs twice daily via systemd timer

## Troubleshooting

### DNS Issues
```bash
# Check DNS propagation
dig +short your-domain.com
nslookup your-domain.com 8.8.8.8
```

### Certificate Issues
```bash
# Check certificate details
openssl x509 -in /etc/letsencrypt/live/your-domain.com/fullchain.pem -text -noout

# Check certificate expiry
certbot certificates
```

### Renewal Issues
```bash
# Check renewal logs
tail -f /var/log/letsencrypt/letsencrypt.log

# Manual renewal test
certbot renew --dry-run
```

## Security Notes

1. **Certificate Files Permissions:** The private key is only readable by root
2. **Auto-Renewal:** Certificates are automatically renewed 30 days before expiry
3. **Docker Access:** Mount certificates as read-only in containers
4. **Backup:** Consider backing up `/etc/letsencrypt/` directory

## Summary

After following this guide, you will have:

✅ SSL certificates installed for your domain
✅ Automatic certificate renewal configured
✅ Nginx removed from the system
✅ Ports 80 and 443 free for Docker containers
✅ Certificates ready to use in Docker/containerized applications

The certificates will automatically renew every 60 days, ensuring your SSL setup remains valid without manual intervention.
