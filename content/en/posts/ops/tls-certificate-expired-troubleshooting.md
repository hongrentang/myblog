---
title: "TLS Certificate Expired — From NET::ERR_CERT_DATE_INVALID to Certificate Lifecycle Management"
date: 2026-05-25
weight: 100170
slug: "tls-certificate-expired-troubleshooting"
tags: ["tls", "certificate", "security", "ingress", "troubleshooting"]
categories: ["安全"]
description: "Website suddenly shows NET::ERR_CERT_DATE_INVALID — debugging from DNS, CDN, and Ingress restarts to finding the expired certificate and fixing cert-manager renewal"
keywords: "tls certificate expired, ssl error, cert-manager, letsencrypt, certificate renewal"
draft: false
featured: true
cover:
  image: "/images/tls-cert-expired-banner.svg"
  caption: "TLS Certificate Expired Troubleshooting"
---

# TLS Certificate Expired — From NET::ERR_CERT_DATE_INVALID to Certificate Lifecycle Management

## The Incident

Monday, 10 AM. Users report the website is broken. Browser shows:

```
NET::ERR_CERT_DATE_INVALID
```

Check with curl:

```bash
curl -v https://www.example.com
```

```
*   Trying 203.0.113.10:443...
* Connected to www.example.com (203.0.113.10) port 443 (#0)
* ALPN: offers h2,http/1.1
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256
* Certificate chain
  *  subject: CN=www.example.com
  *  start date: May 20 2025 00:00:00 GMT
  *  expire date: May 20 2026 00:00:00 GMT
  *  subjectAltName: host "www.example.com" matched
* SSL certificate verify result: 10 (certificate has expired)
* Closing connection
* TLS error: certificate has expired
curl: (60) SSL certificate problem: certificate has expired
```

`certificate has expired` — plain and simple.

The traffic path:

```
User → Cloudflare → Nginx Ingress Controller → Backend Service
```

TLS is terminated at Nginx Ingress with a Let's Encrypt certificate, valid for 1 year.

**Impact**: All HTTPS traffic is down. Users see "Your connection is not private" warnings. API clients fail with SSL errors.

## Investigation

### Wrong turn 1: Blame DNS / CDN

Browser security error — first thought: Cloudflare misconfiguration? DNS resolving to a wrong IP?

```bash
dig +short www.example.com
```

```
203.0.113.10
```

IP is correct.

```bash
curl -I http://www.example.com
```

```
HTTP/1.1 301 Moved Permanently
Location: https://www.example.com/
```

HTTP works fine (301 redirect) — the origin is alive. Problem is HTTPS-only.

**Lesson learned**: HTTP works, HTTPS fails — 90% of the time this is a TLS certificate issue, not DNS or network. HTTP and HTTPS follow different verification paths in the browser. If HTTP works, the network path is fine.

### Wrong turn 2: Check Cloudflare origin certificate

"Maybe Cloudflare's edge certificate expired?"

```
Edge Certificates:
  www.example.com          Valid         2025-05-20 ~ 2026-05-20
  *.example.com            Valid         2025-05-20 ~ 2026-05-20
```

Cloudflare edge certs are fine. But the configuration is **Full (strict)** — Cloudflare also validates the origin's TLS certificate.

```bash
# Bypass CDN, hit origin directly
curl -v https://203.0.113.10 --resolve www.example.com:443:203.0.113.10
```

```
* SSL certificate verify result: 10 (certificate has expired)
```

Origin TLS cert is expired too. Not a Cloudflare issue.

### Wrong turn 3: Restart Ingress Controller

"Maybe a restart will reload the certificate."

```bash
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```

After rollout:

```bash
curl -I https://www.example.com
```

Still `NET::ERR_CERT_DATE_INVALID`.

**Lesson learned**: Restarting Nginx won't renew an expired certificate. The certificate is a static resource stored in a K8S Secret. Restarting the pod just re-reads the same expired cert from the same Secret. It's like putting an expired ID in a new wallet — the ID is still expired.

### The real finding: check the certificate

Examine the Ingress TLS config:

```bash
kubectl describe ingress www-example-com -n production
```

```
TLS:
  hosts:
    - www.example.com
  secretName: www-example-com-tls
```

Decode the certificate:

```bash
kubectl get secret www-example-com-tls -n production -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout | grep -E "Not Before|Not After"
```

```
Not Before: May 20 00:00:00 2025 GMT
Not After : May 20 00:00:00 2026 GMT
```

```bash
kubectl get secret www-example-com-tls -n production -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -enddate -noout -checkend 0
```

```
Certificate will expire
```

Today is May 25, 2026 — expired 5 days ago.

Check cert-manager status:

```bash
kubectl get certificate -n production
```

```
NAME                  READY   SECRET                AGE
www-example-com-tls   False   www-example-com-tls   365d
```

```bash
kubectl describe certificate www-example-com-tls -n production
```

```
Status:
  Conditions:
    Reason:              Expired
    Status:              True
    Type:                Expired
  Not After:             2026-05-20T00:00:00Z
  Renewal Time:          2026-04-20T00:00:00Z
Events:
  Warning  RenewalError  30d   cert-manager  failed to renew: acme: error: 403 :: POST :: Invalid order
```

**RenewalError** — 30 days ago, cert-manager failed to renew. ACME returned 403.

## Root Cause

| Layer | Cause |
|-------|-------|
| Direct | Let's Encrypt certificate expired on 2026-05-20, all HTTPS down |
| Renewal failure | cert-manager ACME renewal returned 403 (DNS-01 challenge failed) |
| Monitoring gap | No certificate expiry alerting or renewal failure notification |
| Why no auto-recovery | cert-manager kept retrying but underlying credentials were expired |

The renewal flow:

```
30 days before expiry
  → cert-manager initiates ACME challenge
  → DNS-01 challenge: needs to write TXT record via Cloudflare API
  → API call fails (403 Forbidden)
  → cert-manager retries but never succeeds
  → certificate stays expired
```

The root cause: **The Cloudflare API Token used for DNS-01 challenge had expired**.

```bash
kubectl describe clusterissuer letsencrypt-prod
```

```
Spec:
  Acme:
    Solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

```bash
kubectl get secret cloudflare-api-token -n cert-manager -o yaml
```

The API Token had a 6-month validity. It expired exactly when cert-manager tried to renew the certificate. cert-manager couldn't modify DNS TXT records to prove domain ownership, so Let's Encrypt refused to issue the new certificate.

## Fix

### Emergency: manually issue a new certificate

```bash
# Option A: certbot with manual DNS challenge
certbot certonly --manual --preferred-challenges dns \
  -d www.example.com -d *.example.com

# Add the TXT record in DNS console, verify
```

```bash
# Create K8S Secret from the new certificate
kubectl create secret tls www-example-com-tls \
  --cert=/etc/letsencrypt/live/www.example.com/fullchain.pem \
  --key=/etc/letsencrypt/live/www.example.com/privkey.pem \
  -n production --dry-run=client -o yaml | kubectl apply -f -
```

```bash
# Verify
curl -I https://www.example.com
# 200 OK
# SSL certificate verify ok
```

### Root fix: update Cloudflare Token and re-trigger renewal

```bash
# 1. Generate new Cloudflare API Token in Cloudflare dashboard
# Permissions: Zone:DNS:Edit

# 2. Update K8S Secret
kubectl delete secret cloudflare-api-token -n cert-manager
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=<new-token> \
  -n cert-manager

# 3. Delete expired certificate resource to force re-issue
kubectl delete certificate www-example-com-tls -n production

# 4. Re-create Certificate
cat <<'EOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: www-example-com-tls
  namespace: production
spec:
  secretName: www-example-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - www.example.com
    - example.com
  renewBefore: 720h
EOF
```

```bash
kubectl get certificate -n production -w
```

```
NAME                  READY   SECRET                AGE
www-example-com-tls   True    www-example-com-tls   1m
```

### Verification

```bash
# 1. Browser test — no security warnings

# 2. curl
curl -vI https://www.example.com 2>&1 | grep "SSL certificate"
# SSL certificate verify ok.

# 3. Check expiration
kubectl get secret www-example-com-tls -n production \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates

# 4. cert-manager status
kubectl get certificate -n production
# READY: True
```

### Long-term prevention

```bash
# 1. Certificate expiry alerting (30d, 7d, 1d before)
# Prometheus Blackbox Exporter
# probe_ssl_earliest_cert_expiry < (30 * 86400)

# Or a simple cron check:
for domain in www.example.com api.example.com; do
  expiry=$(openssl s_client -connect $domain:443 -servername $domain </dev/null 2>/dev/null | openssl x509 -noout -enddate | cut -d= -f2)
  expiry_epoch=$(date -d "$expiry" +%s)
  days_left=$(( (expiry_epoch - $(date +%s)) / 86400 ))
  echo "$domain: $days_left days remaining"
  if [ $days_left -lt 7 ]; then
    echo "ALERT: Certificate for $domain expires in $days_left days!"
  fi
done

# 2. Monitor cert-manager renewal events
# Watch for Certificate RenewalError events
kubectl get events --all-namespaces --field-selector reason=RenewalError

# 3. API Token lifecycle management
# Cloudflare tokens: set to 1-year max, add calendar reminder
# Or use external secrets (Vault, AWS Secrets Manager) with auto-rotation

# 4. Quarterly renewal drill
# Test the full renewal process every 3 months
# Verify both HTTP-01 and DNS-01 challenge paths work
```

## What I Learned

1. **Certificate expiry isn't "if" but "when."** Let's Encrypt's 90-day validity means 4 renewals per year. Every renewal can fail (rate limits, token expiry, domain validation errors). Automation only works when **every link in the chain has monitoring** — cert-manager's renewal attempts, ACME validation results, and the certificate's NotAfter timestamp.

2. **NET::ERR_CERT_DATE_INVALID → check the certificate first.** Don't restart Ingress, don't query DNS, don't call your CDN support. One `openssl s_client` command shows you the certificate dates, issuer, and subject. The worst SSL debugging mistake is spinning wheels in the wrong layer.

3. **cert-manager's RenewalError won't auto-recover if the underlying credential is dead.** If the Cloudflare API Token, AWS Route53 key, or other DNS provider credential expires, cert-manager will retry forever and never succeed. Those external credentials need lifecycle management too. Monitor them, rotate them, alert before they expire.

4. **TLS is security's front door — and the most neglected one.** You have CPU alerts, disk alerts, memory alerts. But certificate expiry often slips through. Add certificate validity monitoring to your stack — Prometheus Blackbox Exporter's `probe_ssl_earliest_cert_expiry` or a simple cron script. Certificate expiry is a single point of failure for your entire HTTPS infrastructure, and when it goes, ALL ports go at once.
