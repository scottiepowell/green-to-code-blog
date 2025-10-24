---
layout: post
title: "NFSv4 with Kerberos and FreeIPA in the ROADMAPS.LINK Environment"
date: 2025-05-14
author: "Auto Writer"
tags: [NFS, Kerberos, FreeIPA, ROADMAPS.LINK]
nav_order: 4
parent: The IPA
grand_parent: Articles
---

# NFSv4 with Kerberos and FreeIPA in the ROADMAPS.LINK Environment

> Deploy a FreeIPA-backed Kerberos realm for NFSv4 so that every file share mount, read, and write in ROADMAPS.LINK is cryptographically authenticated, optionally integrity-protected, and encrypted.

## Overview

Network File System (NFS) version 4 paired with Kerberos is a cornerstone technology for secure, centralized file sharing in the ROADMAPS.LINK environment. By federating authentication and authorization through FreeIPA, every read, write, and delete on an exported share is verified against a trusted Kerberos principal. Administrators can select from three graduated security profiles—`sec=krb5`, `sec=krb5i`, and `sec=krb5p`—to balance performance with integrity and privacy requirements.

## Why Kerberos-Backed NFS Matters

Traditional NFS deployments rely on static UID/GID mappings or IP-based trust, which are difficult to scale and audit. Integrating NFSv4 with Kerberos eliminates shared secrets and root squashing while enforcing per-user accountability. Each access is tied to a valid service ticket issued by FreeIPA, enabling:

- **Strong authentication** (`sec=krb5`): ensures clients present verifiable Kerberos identities before mounting.
- **Integrity protection** (`sec=krb5i`): adds checksums that prevent tampering in transit.
- **Confidentiality** (`sec=krb5p`): encrypts traffic for workloads with sensitive data.

## Prerequisites

### DNS and Time Synchronization

Consistency in name resolution and timekeeping is non-negotiable for Kerberos. Confirm that all hosts—`nfs1.roadmaps.link`, `cli1.roadmaps.link`, and your FreeIPA controllers—resolve each other’s FQDNs. Deploy Chrony or NTP to keep clocks aligned:

```bash
sudo dnf -y install chrony   # or sudo apt -y install chrony
sudo systemctl enable --now chronyd
chronyc tracking
```

### Kerberos Service Principals and Keytabs

Each NFS participant requires its own Kerberos service principal and a synchronized keytab. Register the services once per host in FreeIPA, then fetch the keys locally:

```bash
ipa service-add nfs/nfs1.roadmaps.link
ipa service-add nfs/cli1.roadmaps.link

sudo ipa-getkeytab -s ipa1.roadmaps.link \
  -p nfs/nfs1.roadmaps.link@ROADMAPS.LINK \
  -k /etc/krb5.keytab -r

sudo ipa-getkeytab -s ipa1.roadmaps.link \
  -p nfs/cli1.roadmaps.link@ROADMAPS.LINK \
  -k /etc/krb5.keytab -r

klist -k /etc/krb5.keytab | grep nfs/
```

### Identity Mapping Consistency

NFSv4 depends on unified ID mapping so that POSIX ownership stays predictable across hosts. Configure `/etc/idmapd.conf` with a common domain and restart the mapping daemon when necessary:

```ini
[General]
Domain = roadmaps.link
```

```bash
sudo systemctl restart nfs-idmapd
```

## Provisioning the NFSv4 Server

Whether you run RHEL derivatives or Ubuntu, the workflow remains consistent.

1. **Install dependencies.**
   ```bash
   sudo dnf -y install nfs-utils    # RHEL/Rocky/Alma
   sudo apt -y install nfs-kernel-server
   ```
2. **Enable core services.**
   ```bash
   sudo systemctl enable --now nfs-server
   sudo systemctl enable --now rpc-gssd rpc-svcgssd 2>/dev/null || true
   ```
3. **Define Kerberos-enforced exports.** Add the share to `/etc/exports`:
   ```
   /srv/projects  10.30.0.0/24(rw,sec=krb5p,no_subtree_check)
   ```
4. **Publish the export.**
   ```bash
   sudo exportfs -rav
   sudo systemctl restart nfs-server
   exportfs -v
   ```

With `sec=krb5p`, data is encrypted end-to-end while retaining the convenience of NFS semantics for the entire `10.30.0.0/24` subnet.

## Preparing NFSv4 Clients

1. **Install the client stack.**
   ```bash
   sudo dnf -y install nfs-utils    # RHEL/Rocky/Alma
   sudo apt -y install nfs-common
   sudo systemctl enable --now rpc-gssd 2>/dev/null || true
   ```
2. **Obtain Kerberos credentials.**
   ```bash
   kinit alice@ROADMAPS.LINK
   klist
   ```
3. **Mount the share with Kerberos privacy.**
   ```bash
   sudo mkdir -p /mnt/projects
   sudo mount -t nfs4 -o sec=krb5p nfs1.roadmaps.link:/srv/projects /mnt/projects
   mount | grep projects
   ```

When the mount succeeds, the session ticket for `alice@ROADMAPS.LINK` underpins authorization decisions, ensuring the file system enforces your FreeIPA policies.

## Validating Access Control

Leverage FreeIPA-managed POSIX identities to confirm that permissions translate correctly across the environment:

```bash
# As Alice
id -u; id -Gn
sudo -u alice touch /mnt/projects/alice_was_here

# As Bob (different group memberships)
sudo -u bob -H bash -lc 'touch /mnt/projects/bob_was_here; ls -l /mnt/projects'
```

Matching ownership and group metadata demonstrates that SSSD, idmapd, and Kerberos remain in sync.

## Operational Best Practices

- **Stay consistent.** Keep the `Domain = roadmaps.link` directive synchronized on every node.
- **Manage ticket lifecycles.** Renew expiring credentials (`kinit -R`) or extend lifetimes in `/etc/krb5.conf` with `renew_lifetime = 7d` for prolonged transfers.
- **Troubleshoot identity drift.** If UIDs or GIDs appear misaligned, validate identity resolution with `getent passwd <user>` and `getent group <group>`, then restart `sssd` or `rpc.idmapd` if required.

## Summary

Deploying NFSv4 with Kerberos inside ROADMAPS.LINK elevates file sharing from best effort to security-first. Selecting the appropriate `sec=` level gives you precise control over authentication, integrity, and privacy, while FreeIPA centralizes policy management for both users and systems. The result is a scalable, auditable, and compliant storage fabric that empowers multi-host collaboration without sacrificing control.

[Read the full article on NFSv4 with Kerberos in the ROADMAPS.LINK environment](https://green-to-code-blog.roadmaps.link/articles/2025-05-14-nfsv4-kerberos-freeipa-roadmaps-link)

