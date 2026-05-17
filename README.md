# Rocky Linux Network Infrastructure Deployment

> End-to-end provisioning and hardening of a production-grade network services stack on Rocky Linux, covering authoritative DNS, encrypted mail delivery, and TLS-enforced web hosting — all verified from a live Ubuntu client.

---

## Overview

This project documents the full lifecycle deployment of four core network services on a Rocky Linux server within an isolated VM lab environment (`bonbon.net` domain, `192.168.200.0/24` subnet). Each service was configured from bare packages through to client-verified connectivity, with an emphasis on encrypted transport, firewall hygiene, and service persistence across reboots.

**Server OS:** Rocky Linux  
**Client OS:** Ubuntu Linux  
**Domain:** `bonbon.net`  
**Server IP:** `192.168.200.4` | **Client IP:** `192.168.200.5`

---

## Table of Contents

- [Task 1 — DNS (BIND)](#task-1--authoritative-dns-with-bind)
- [Task 2 — Secure Email (Postfix + Dovecot + SSL/TLS)](#task-2--secure-mail-delivery-postfix--dovecot)
- [Task 3 — HTTPS Web Server (Apache + mod_ssl)](#task-3--apache-web-server-with-tlsssl)
- [Architecture Notes](#architecture-notes)

---

## Task 1 — Authoritative DNS with BIND

### Objective
Deploy a fully authoritative DNS server capable of forward and reverse resolution for the internal `bonbon.net` zone, and recursive resolution for external domains.

### Installation

```bash
sudo dnf install bind bind-utils -y
```

### Core Configuration — `/etc/named.conf`

The global `options` block was modified to accept queries on all interfaces and from any source, enabling the Ubuntu client to reach the resolver:

```bind
options {
    listen-on port 53 { any; };
    allow-query     { any; };
};
```

Zone declarations were appended to register the forward and reverse zones with the `named` daemon:

```bind
zone "bonbon.net" IN {
    type master;
    file "fwd.bonbon.net.db";
    allow-update { none; };
};

zone "200.168.192.in-addr.arpa" IN {
    type master;
    file "rvs.200.168.192.db";
    allow-update { none; };
};
```

### Forward Zone File — `/var/named/fwd.bonbon.net.db`

```dns
$TTL 86400
@   IN  SOA  server.bonbon.net. root.bonbon.net. (
        2025112401 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@      IN  NS   server.bonbon.net.
server IN  A    192.168.200.4
client IN  A    192.168.200.5
```

### Reverse Zone File — `/var/named/rvs.200.168.192.db`

```dns
$TTL 86400
@   IN  SOA  server.bonbon.net. root.bonbon.net. (
        2025112401 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@  IN  NS   server.bonbon.net.
4  IN  PTR  server.bonbon.net.
5  IN  PTR  client.bonbon.net.
```

### Service Activation & Firewall

File ownership was transferred to the `named` group before service start to ensure the daemon can read the zone files at runtime:

```bash
sudo chown root:named /var/named/fwd.bonbon.net.db /var/named/rvs.200.168.192.db

sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload

sudo systemctl enable --now named
sudo systemctl status named
```

### Verification

**Server-side (Rocky Linux):**
```bash
nslookup server.bonbon.net   # Forward lookup → 192.168.200.4
nslookup 192.168.200.4       # Reverse lookup → server.bonbon.net
```

**Client-side (Ubuntu):**  
The Ubuntu client's DNS resolver was pointed to `192.168.200.4` via manual IPv4 settings. The following queries were validated from the Ubuntu terminal:

```bash
nslookup server.bonbon.net   # Resolves to 192.168.200.4
nslookup client.bonbon.net   # Resolves to 192.168.200.5
nslookup google.com          # Confirms external recursive resolution
```

All three queries returned correct results, confirming the BIND deployment handles both authoritative internal resolution and recursive external forwarding.

---

## Task 2 — Secure Mail Delivery (Postfix + Dovecot)

### Objective
Deploy a multi-component mail stack using Postfix (MTA) and Dovecot (MDA), enforce TLS encryption end-to-end using a self-signed certificate, and validate delivery from the Ubuntu client using Thunderbird over IMAPS/SMTPS.

### Installation

```bash
sudo dnf install postfix dovecot -y
```

### Postfix Configuration — `/etc/postfix/main.cf`

Key parameters were set to define the server's identity, bind to all interfaces, scope trusted relay networks, and route mail to Maildir-format storage compatible with Dovecot:

```ini
# Server identity
myhostname = server.bonbon.net
mydomain   = bonbon.net
myorigin   = $mydomain

# Network binding and relay scope
inet_interfaces  = all
inet_protocols   = all
mydestination    = $myhostname, localhost.$mydomain, localhost, $mydomain
mynetworks       = 192.168.200.0/24, 127.0.0.0/8

# Maildir storage (Dovecot-compatible)
home_mailbox = Maildir/
```

### Dovecot Configuration

**Protocol enablement** — `/etc/dovecot/dovecot.conf`:
```ini
protocols = imap pop3 lmtp submission
```

**Maildir storage path** — `/etc/dovecot/conf.d/10-mail.conf`:
```ini
mail_location = maildir:~/Maildir
```

### TLS Certificate Generation

A self-signed X.509 certificate with a 365-day validity was generated and saved to the standard PKI directories:

```bash
sudo openssl req -new -x509 -days 365 -nodes \
  -out /etc/pki/tls/certs/mail.cert \
  -keyout /etc/pki/tls/private/mail.key \
  -subj "/C=MY/ST=Selangor/L=KL/O=Bonbon/OU=IT/CN=server.bonbon.net"
```

### Dovecot SSL Enforcement — `/etc/dovecot/conf.d/10-ssl.conf`

```ini
ssl      = required
ssl_cert = </etc/pki/tls/certs/mail.cert
ssl_key  = </etc/pki/tls/private/mail.key
```

Setting `ssl = required` instructs Dovecot to reject any plaintext authentication attempt, enforcing IMAPS/POP3S-only access.

### Postfix SMTPS — `/etc/postfix/master.cf`

The `smtps` block was uncommented and configured to activate secure mail submission on port 465 with SASL authentication and TLS wrapping.

### Firewall — Secure Mail Ports

```bash
sudo firewall-cmd --permanent --add-service={smtps,imaps,pop3s}
sudo firewall-cmd --reload
```

### Client Validation — Mozilla Thunderbird (Ubuntu)

Thunderbird was configured with manual server settings to align with the hardened server:

| Direction | Protocol | Server            | Port | Security  |
|-----------|----------|-------------------|------|-----------|
| Incoming  | IMAP     | `192.168.200.4`   | 993  | SSL/TLS   |
| Outgoing  | SMTP     | `192.168.200.4`   | 465  | SSL/TLS   |

A test email was successfully sent and received within the same account, confirming that both Postfix (SMTPS) and Dovecot (IMAPS) are operational and that the self-signed certificate is being applied to all sessions.

---

## Task 3 — Apache Web Server with TLS/SSL

### Objective
Deploy and harden an Apache HTTP server to serve web content exclusively over HTTPS using a self-signed certificate, with automatic HTTP-to-HTTPS redirection enforced at the virtual host level.

### Installation

```bash
sudo dnf install httpd -y
sudo systemctl enable --now httpd
```

### Firewall — Web Traffic

```bash
sudo firewall-cmd --permanent --add-service={http,https}
sudo firewall-cmd --reload
```

### Sample Site

A test HTML page was placed in the Apache web root to confirm content serving before enabling TLS:

```bash
# /var/www/html/index.html
echo "<h1>bonbon.net — Apache on Rocky Linux</h1>" | sudo tee /var/www/html/index.html
```

### SSL/TLS Setup

```bash
sudo dnf install mod_ssl openssl -y
```

A self-signed certificate was generated and its paths were referenced in `/etc/httpd/conf.d/ssl.conf`, replacing the default `localhost` certificate entries:

```apache
SSLCertificateFile    /etc/pki/tls/certs/server.crt
SSLCertificateKeyFile /etc/pki/tls/private/server.key
```

Apache was restarted to apply the updated TLS configuration:

```bash
sudo systemctl restart httpd
```

### HTTP → HTTPS Redirection

An automatic redirect was configured within the HTTP `VirtualHost` block to ensure all plaintext requests are transparently promoted to HTTPS:

```apache
<VirtualHost *:80>
    ServerName server.bonbon.net
    Redirect permanent / https://server.bonbon.net/
</VirtualHost>
```

### Client Verification (Ubuntu)

HTTPS access was confirmed from the Ubuntu client browser by navigating to `https://192.168.200.4`. A browser security prompt was presented due to the self-signed certificate — expected and accepted behavior in a private CA environment. The site loaded successfully after proceeding through the certificate exception, confirming encrypted transport was active.

---

## Architecture Notes

```
┌─────────────────────────────────────────────────┐
│              VM Lab Network                     │
│            192.168.200.0/24                     │
│                                                 │
│   ┌──────────────────────┐                      │
│   │   Rocky Linux Server  │  .200.4             │
│   │  ─────────────────── │                      │
│   │  BIND (DNS)   :53    │                      │
│   │  Postfix (MTA):25/465│                      │
│   │  Dovecot (MDA):993   │                      │
│   │  Apache (HTTPS):443  │                      │
│   └──────────┬───────────┘                      │
│              │                                  │
│   ┌──────────┴───────────┐                      │
│   │   Ubuntu Client       │  .200.5             │
│   │  ─────────────────── │                      │
│   │  DNS resolver → .4   │                      │
│   │  Thunderbird (MUA)   │                      │
│   │  Browser (HTTPS)     │                      │
│   └──────────────────────┘                      │
└─────────────────────────────────────────────────┘
```

| Service    | Package(s)          | Protocol(s)        | Port(s)       | Encryption        |
|------------|---------------------|--------------------|---------------|-------------------|
| DNS        | `bind`, `bind-utils`| DNS/UDP+TCP        | 53            | —                 |
| Mail (MTA) | `postfix`           | SMTP / SMTPS       | 25 / 465      | TLS (self-signed) |
| Mail (MDA) | `dovecot`           | IMAP / IMAPS       | 143 / 993     | TLS (self-signed) |
| Web        | `httpd`, `mod_ssl`  | HTTP → HTTPS       | 80 → 443      | TLS (self-signed) |

---

*Deployed on Rocky Linux 9 · Verified against Ubuntu 22.04 LTS client · Academic project — APD2F2509CS(CYB)*
