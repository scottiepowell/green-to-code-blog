---
layout: post
title: "Kerberos 101 in FreeIPA"
date: 2025-05-11
author: "Auto Writer"
tags: [FreeIPA, Kerberos, Authentication]
nav_order: 1
parent: The IPA Series
grand_parent: Articles
---

# Kerberos 101 in FreeIPA

> Fast-track your FreeIPA authentication knowledge with a quick walkthrough of Kerberos concepts, configuration, and troubleshooting commands that keep single sign-on humming.

## Key Concepts at a Glance

- **TGT (Ticket-Granting Ticket):** Issued by the KDC as `krbtgt/EXAMPLE.LAB` and used to request access to services.
- **TGS (Service Ticket):** Service-specific credentials such as `HTTP/app01.example.lab@EXAMPLE.LAB` obtained after presenting a valid TGT.
- **Realm ↔ DNS Mapping:** Uppercase Kerberos realms map to lowercase DNS domains (for example, `EXAMPLE.LAB` ↔ `example.lab`).
- **Tight Time Synchronization:** Kerberos allows only a small amount of clock skew (≤5 minutes by default), so enforce NTP everywhere.

## Minimal `krb5.conf` That Works

Place the following on clients and servers for a lab-friendly configuration:

```ini
[libdefaults]
  default_realm = EXAMPLE.LAB
  rdns = false
  dns_lookup_kdc = true
  dns_lookup_realm = true
  ticket_lifetime = 10h
  renew_lifetime = 7d
  forwardable = true
  udp_preference_limit = 0

[domain_realm]
  .example.lab = EXAMPLE.LAB
  example.lab = EXAMPLE.LAB
```

## Enforce NTP/Chrony Everywhere

Install and enable chrony on every system so clock skew never gets a chance to break Kerberos:

```bash
# RHEL/Rocky/Alma
sudo dnf -y install chrony
sudo systemctl enable --now chronyd

# Ubuntu
sudo apt -y install chrony
sudo systemctl enable --now chrony
```

Point all systems to the same authoritative time source, or use IPA itself if you have enabled its NTP service.

## Everyday Kerberos Commands

Keep these commands handy for routine Kerberos hygiene:

```bash
# get a TGT for the current user
kinit                            # prompts for password
klist                            # show tickets
kdestroy                         # drop all tickets
# get a TGT as a specific principal
kinit alice@EXAMPLE.LAB
# renew if still valid
kinit -R
```

## Troubleshooting Essentials

Start with these tools when something feels off:

```bash
# check what the KDC is returning
kvno host/app01.example.lab
# show local keytabs (root)
klist -k /etc/krb5.keytab
# client SSSD/Kerberos diagnostics
sssctl domain-status -o
journalctl -u sssd --since -1h
```

## Where Kerberos Commonly Breaks

- **Time Skew:** Produces “Clock skew too great.” Verify with `chronyc sources -v` and fix chrony or NTP first.
- **Wrong DNS or SRV Records:** Leads to “Cannot find KDC.” Use `dig _kerberos._udp.example.lab SRV +short` to confirm service discovery.
- **Missing TGTs:** Services such as SSH or HTTP/SPNEGO fall back or fail silently when no ticket exists. Always `kinit` before testing.

---

Continue exploring FreeIPA authentication in [The IPA series overview](/articles/the-ipa/).
