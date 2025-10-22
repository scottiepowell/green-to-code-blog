---
layout: post
title: "Locking Down FreeIPA Firewall Rules on AlmaLinux"
date: 2025-05-06
author: "Auto Writer"
tags: [FreeIPA, AlmaLinux, Firewall, Security]
nav_order: 3
parent: Articles
---

# Locking Down FreeIPA Firewall Rules on AlmaLinux

> A practical guide to configuring AlmaLinux firewalld so a single FreeIPA server at `192.168.1.20/24` only exposes the services it truly needs.

## Why Minimal Firewall Rules Matter

FreeIPA bundles a full identity management stack (Kerberos, LDAP, DNS, and certificate services). Exposing more network services than necessary increases the attack surface, especially when you run a single server without replicas. Tightening the firewall to only the required protocols ensures that domain members, administrators, and DNS resolvers can reach the identity platform while blocking everything else.

## Assumptions

- AlmaLinux 9 (or 8) server with `firewalld` enabled and running.
- Static IP address `192.168.1.20` with DNS records pointing to the host (e.g., `ipa.example.test`).
- No additional FreeIPA replicas—this guide targets a standalone server.
- Clients and administrative workstations are on networks that should reach the FreeIPA services listed below.

Verify the firewall daemon is active before proceeding:

```bash
sudo systemctl status firewalld --no-pager
```

If `firewalld` is inactive, enable it and confirm the default zone:

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --get-default-zone
```

## Allow-Only-What-Is-Needed Port Matrix

The following table captures the minimal ingress requirements for a FreeIPA server when you expose only core identity features.

| Purpose | Protocol | Port(s) | Notes |
| --- | --- | --- | --- |
| Web UI / enrollment helpers | TCP | 80, 443 | Required for the web console and client bootstrapping scripts. |
| LDAP directory | TCP | 389 | Core directory access. |
| LDAPS directory | TCP | 636 | Optional but recommended for encrypted LDAP binds. |
| Kerberos KDC | TCP/UDP | 88 | Ticket granting (TCP for large tickets, UDP for small). |
| Kerberos password change | TCP/UDP | 464 | Handles password and key changes. |
| DNS (if using integrated DNS) | TCP/UDP | 53 | Authoritative responses and zone transfers. |
| NTP (if using IPA for NTP) | UDP | 123 | Optional time service for Kerberos accuracy. |

> **Tip:** Modern distributions include aggregate firewalld services like `freeipa-4` that bundle multiple ports, reducing manual port management.

## Configuring Firewalld Using Service Names

Firewalld service definitions simplify maintenance by grouping protocol/port combinations. Apply the following commands in the default zone (usually `public`) to open only the required FreeIPA services:

```bash
sudo firewall-cmd --permanent --add-service=freeipa-4
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --permanent --add-service=ntp
sudo firewall-cmd --permanent --add-service=freeipa-replication
sudo firewall-cmd --reload
```

- `freeipa-4` covers HTTP/HTTPS, LDAP/LDAPS, and Kerberos (including password change).
- `dns` is necessary if your FreeIPA server answers authoritative DNS queries or handles dynamic updates.
- `ntp` is optional—enable it only if clients sync time from the FreeIPA host.
- `freeipa-replication` is optional on a single-server topology but provides clarity should you later add replicas.

Confirm the rules took effect:

```bash
sudo firewall-cmd --list-services
```

## Using Explicit Port Rules (Alternative Approach)

If you prefer raw ports or your AlmaLinux release lacks the `freeipa-4` service, add each port explicitly:

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=389/tcp
sudo firewall-cmd --permanent --add-port=636/tcp
sudo firewall-cmd --permanent --add-port=88/tcp
sudo firewall-cmd --permanent --add-port=88/udp
sudo firewall-cmd --permanent --add-port=464/tcp
sudo firewall-cmd --permanent --add-port=464/udp
sudo firewall-cmd --permanent --add-port=53/tcp
sudo firewall-cmd --permanent --add-port=53/udp
sudo firewall-cmd --permanent --add-port=123/udp
sudo firewall-cmd --reload
```

Review the runtime rules after reloading to ensure they match expectations:

```bash
sudo firewall-cmd --list-ports
```

## Testing Connectivity from Clients

Once the firewall is configured, validate access from a client workstation joined to the `192.168.1.0/24` network:

```bash
# HTTP/HTTPS reachability for the web console
curl -Ik http://ipa.example.test
curl -Ik https://ipa.example.test

# Kerberos ticket acquisition
kinit admin
klist

# DNS query if FreeIPA provides DNS
dig @192.168.1.20 ipa.example.test A
```

If clients rely on the FreeIPA server for NTP, confirm time synchronization:

```bash
chronyc sources -v
```

## Hardening Tips

- Restrict management access (`ssh`, `ipa` CLI) to trusted admin networks using additional firewalld zones.
- Leverage `--set-target=DROP` on unused zones to silently discard unexpected traffic.
- Schedule periodic reviews of `sudo firewall-cmd --list-all-zones` to ensure no temporary rules remain enabled.
- Combine firewall restrictions with SELinux in enforcing mode for defense-in-depth.

## Conclusion

By limiting the FreeIPA server to the essential protocols defined in the allow-only-what-is-needed matrix, AlmaLinux firewalld enforces a tight perimeter that supports client enrollment, directory lookups, Kerberos authentication, and (optionally) DNS and NTP—all without exposing superfluous services.

## View This Article

Once published, access the guide at:

```
https://green-to-code-blog.roadmaps.link/articles/2025-05-06-freeipa-firewall-hardening.html
```

Bookmark the link for future reference when auditing your FreeIPA deployment.
