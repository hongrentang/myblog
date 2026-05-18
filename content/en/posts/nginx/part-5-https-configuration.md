---
title: "Nginx from Beginner to Pro · Part 5: HTTPS Configuration"
date: 2025-01-09
weight: 1105
draft: false
tags: ["nginx"]
---

## Why HTTPS?

- **Encryption**: Prevents man-in-the-middle eavesdropping and tampering
- **Authentication**: Confirms the website you're visiting is genuine
- **SEO**: Google gives ranking boosts to HTTPS sites
- **Browser marking**: HTTP sites are marked as "Not Secure"
- **Compliance**: Payment and privacy data require HTTPS

---

## Obtaining SSL/TLS Certificates

### Let's Encrypt (Free, Recommended)

Use Certbot to automatically obtain and renew:

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Obtain and auto-configure Nginx
sudo certbot --nginx -d example.com -d www.example.com

# Obtain certificate only (manual config)
sudo certbot certonly --nginx -d example.com

# View certificates
sudo certbot certificates

# Test renewal
sudo certbot renew --dry-run
```

### Auto-Renewal

```bash
# Add cron job, checks twice daily
echo "0 */12 * * * /usr/bin/certbot renew --quiet" | sudo tee -a /etc/crontab
```

Certbot adds a systemd timer by default, so manual cron is usually unnecessary.

---

## Basic HTTPS Configuration

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/example;
    index index.html;

    # Other config...
}

# HTTP auto-redirect to HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}
```

---

## SSL Configuration Optimization

### Protocol Versions

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

**Do not** include TLSv1.0 or TLSv1.1 — they are deprecated and vulnerable. TLSv1.3 is the latest and most secure protocol.

### Cipher Suites

```nginx
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

ssl_prefer_server_ciphers on;
```

Use the [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) for modern browser-compatible configs.

### DH Parameters

```nginx
ssl_dhparam /etc/nginx/dhparam.pem;
```

Generate DH parameters (takes a few minutes):

```bash
sudo openssl dhparam -out /etc/nginx/dhparam.pem 2048
```

> 2048-bit balances security and performance. 4096-bit is more secure but slower for both generation and handshake.

### HSTS

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

Tells browsers: for the next year, this domain is **only** accessible via HTTPS.

**Warning**: Before enabling HSTS, make sure HTTPS is fully working — otherwise you'll lock users out of your site.

### Full HTTPS Optimization

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # Certificates
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_dhparam         /etc/nginx/dhparam.pem;

    # Protocol and ciphers
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;

    # Session cache (performance)
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    root /var/www/example;
    index index.html;
}
```

---

## OCSP Stapling

OCSP (Online Certificate Status Protocol) lets browsers check if a certificate has been revoked.

Without OCSP Stapling, browsers must make an additional request to the CA's OCSP server, slowing down the handshake.

OCSP Stapling lets Nginx cache the certificate status and return it directly during the TLS handshake:

```nginx
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
```

---

## Multi-Domain HTTPS

Use SNI (Server Name Indication) to host multiple HTTPS sites on a single IP:

```nginx
server {
    listen 443 ssl http2;
    server_name site1.com;
    ssl_certificate     /etc/letsencrypt/live/site1.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/site1.com/privkey.pem;
}

server {
    listen 443 ssl http2;
    server_name site2.com;
    ssl_certificate     /etc/letsencrypt/live/site2.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/site2.com/privkey.pem;
}
```

> Old Windows XP/IE doesn't support SNI and will match the first server block. If you still support these clients, make that block serve a generic certificate.

---

## Security Hardening

### Testing Tools

After deploying HTTPS, always run security tests:

**Online test**: [https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/)

**Command line**:

```bash
# View certificate info
openssl s_client -connect example.com:443 -servername example.com

# Test protocol support
nmap --script ssl-enum-ciphers -p 443 example.com
```

### Rating Target

Qualys SSL Labs score ≥ **A** is the baseline requirement, aim for **A+**.

### Common Security Headers

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "0" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

> Note: Some of these headers are obsolete or replaced by default behavior in modern browsers. Use Content-Security-Policy (CSP) as a more robust alternative.

---

## Performance Optimization

### Session Reuse

SSL/TLS handshakes are slow (especially with RSA key exchange). Enabling session reuse significantly reduces handshake count:

```nginx
ssl_session_cache shared:SSL:10m;    # 10MB shared cache, ~40000 sessions
ssl_session_timeout 1d;              # Cache sessions for 1 day
ssl_session_tickets off;             # Disable session tickets
```

### HTTP/2

```nginx
listen 443 ssl http2;
```

Just add the `http2` parameter. Benefits:

- Multiplexing: multiple resources in one connection
- Header compression: HPACK reduces overhead
- Server Push (deprecated): server pushes resources proactively

---

## Common Issues

### Expired Certificate

```bash
# Check certificate expiry
openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem -noout -dates

# Check days remaining
openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem -noout -enddate
```

Let's Encrypt certificates are valid for 90 days — ensure auto-renewal is working.

### Incomplete Certificate Chain

Some Android devices may lack intermediate certificates. Configure the full chain:

```nginx
# fullchain.pem includes the complete chain (recommended)
ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;

# Or manually concatenate
ssl_certificate /path/to/cert.pem;
ssl_trusted_certificate /path/to/chain.pem;
```

### Mixed Content

Page loaded over HTTPS references HTTP resources (images, scripts) — browsers will block them.

**Check**: Browser DevTools → Console shows `Mixed Content` errors.

**Fix**: Ensure all resources use HTTPS or protocol-relative URLs `//example.com/image.png`.

---

## Summary

HTTPS is now standard for all websites. This article covered the full HTTPS deployment process from certificate acquisition, basic configuration, security optimization to performance tuning.

---

