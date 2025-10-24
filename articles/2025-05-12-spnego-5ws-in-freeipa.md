---
layout: post
title: "5Ws of Apache/Nginx SPNEGO in FreeIPA"
date: 2025-05-12
author: "Auto Writer"
tags: [FreeIPA, SPNEGO, Kerberos, Apache, Nginx]
nav_order: 2
parent: The IPA Series
grand_parent: Articles
---

# 5Ws of Apache/Nginx SPNEGO in FreeIPA

> Understand who should deploy SPNEGO, what it delivers, and how to configure Apache or Nginx with FreeIPA-backed Kerberos single sign-on.

## Who Needs This?

- **Administrators** securing internal dashboards, intranet portals, or Keycloak frontends that already rely on FreeIPA-managed identities.
- **Users** who authenticate within the `ROADMAPS.LINK` realm and expect seamless browser-based single sign-on.

## What Is SPNEGO Doing?

- **Negotiated Kerberos Auth:** Browsers present the `HTTP/app01.roadmaps.link@ROADMAPS.LINK` Kerberos service ticket instead of prompting for credentials.
- **FreeIPA-Managed Keytabs:** The HTTP service principal lives in FreeIPA, while Apache or Nginx reads its keys from a local keytab.
- **No Passwords in Transit:** Kerberos tickets provide mutual authentication and session security without sending reusable passwords.

## When to Turn It On

Deploy SPNEGO when the environment already checks these boxes:

- ✅ FreeIPA-issued Kerberos tickets (`kinit` works everywhere users log in).
- ✅ Functional forward and reverse DNS between clients and servers.
- ✅ Tight NTP/Chrony synchronization so Kerberos timestamps stay valid.
- ✅ HTTPS endpoints on trusted internal domains such as `*.roadmaps.link`.

## Where Each Component Lives

- **FreeIPA Servers:** Issue the HTTP service principal and store its key version numbers (kvnos).
- **Web Servers:** Apache or Nginx hosts keep a copy of the keytab with the HTTP principal.
- **Clients:** Linux, macOS, and modern browsers obtain Kerberos tickets and negotiate SSO with the web tier.

## Why SPNEGO Matters

- Eliminates password prompts for internal applications and management consoles.
- Aligns with centralized identity policies enforced in FreeIPA.
- Provides drop-in Kerberos security for Apache, Nginx (with plugins), and proxy integrations.

---

## Service Principals & Keytabs (Apache/Nginx SPNEGO)

The HTTP service principal anchors Kerberos web authentication. Apache or Nginx must read a keytab that holds the matching secrets.

### Create Service Principals in FreeIPA

```bash
# HTTP service for the web host
ipa service-add HTTP/app01.roadmaps.link

# Host service (usually created automatically during enrollment; add if missing)
ipa service-add host/app01.roadmaps.link
```

### Safely Fetch Keytabs (On the Web Host)

Best practice: pull keytabs directly on the target web host as `root`, then lock the file to the daemon user. Apache/httpd expects a root-owned file with permissions set to `600`.

```bash
sudo ipa-getkeytab -s ipa1.roadmaps.link \
  -p HTTP/app01.roadmaps.link@ROADMAPS.LINK \
  -k /etc/httpd/httpd.keytab \
  -r  # replace existing keys; increments kvno

sudo chmod 600 /etc/httpd/httpd.keytab
sudo chown root:root /etc/httpd/httpd.keytab
klist -k /etc/httpd/httpd.keytab
```

### Rotate Keys Without Downtime

Rotating keys with `-r` bumps the key version number (kvno). Clients negotiate seamlessly as long as they pull new tickets.

```bash
sudo ipa-getkeytab -s ipa1.roadmaps.link \
  -p HTTP/app01.roadmaps.link@ROADMAPS.LINK \
  -k /etc/httpd/httpd.keytab -r
klist -k /etc/httpd/httpd.keytab  # confirm kvno increment
```

## Apache + SPNEGO with `mod_auth_gssapi`

### Install Packages

```bash
# RHEL/Rocky/Alma
sudo dnf -y install httpd mod_auth_gssapi

# Ubuntu/Debian
sudo apt -y install apache2 libapache2-mod-auth-gssapi
sudo a2enmod auth_gssapi
sudo systemctl restart apache2
```

### Configure the Protected VirtualHost

Create `/etc/httpd/conf.d/spnego.conf` (or `/etc/apache2/conf-available/spnego.conf` on Debian-based systems):

```apache
<VirtualHost *:80>
  ServerName app01.roadmaps.link
  # If TLS terminates elsewhere, front this with :443 and proxy; otherwise serve HTTPS here.

  <Location "/protected">
    AuthType GSSAPI
    AuthName "Kerberos login"
    GssapiCredStore keytab:/etc/httpd/httpd.keytab
    GssapiUseSessions On
    Require valid-user
  </Location>
</VirtualHost>
```

Enable and start Apache:

```bash
sudo systemctl enable --now httpd   # or apache2
```

### Test from a Kerberos Client

```bash
kinit alice@ROADMAPS.LINK
# curl with Negotiate
curl --negotiate -u : -I http://app01.roadmaps.link/protected
# Or use a Kerberos-capable browser (ensure `roadmaps.link` is allowed for Negotiate)
```

Expect a `401` with `WWW-Authenticate: Negotiate` followed by a successful `200 OK`. Browsers handle this exchange automatically once `roadmaps.link` is trusted for negotiation.

## Nginx Note

Nginx lacks native SPNEGO, so you have two options:

1. Compile it with `ngx_http_auth_spnego` support; or
2. Place Nginx behind Apache or Keycloak, letting those services perform Kerberos auth and forward headers.

For clarity in this guide, demonstrate SPNEGO on Apache first, then let Nginx handle load balancing or reverse proxy duties behind it.

## Key Takeaway

Pairing the `HTTP/app01.roadmaps.link@ROADMAPS.LINK` service principal with SPNEGO enables seamless, password-less SSO for internal FreeIPA-integrated web applications.

---

Continue exploring FreeIPA authentication in [The IPA series overview](/articles/the-ipa/).
