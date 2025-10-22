---
layout: post
title: "How to Install a FreeIPA Server on AlmaLinux"
date: 2025-05-05
author: "Auto Writer"
tags: [FreeIPA, AlmaLinux, Identity Management, Linux]
---

# How to Install a FreeIPA Server on AlmaLinux

> A step-by-step guide for deploying a FreeIPA identity management server on AlmaLinux with the host configured for the `192.168.1.20/24` network.

## Introduction

FreeIPA provides centralized identity, policy, and audit services by integrating technologies such as 389 Directory Server, Kerberos, DNS, and Dogtag Certificate System. This guide walks through preparing an AlmaLinux host and installing the FreeIPA server package so that your infrastructure gains unified authentication and host management capabilities.

## Prerequisites

- AlmaLinux 9 (or 8) server with root or sudo access.
- Static IP configured as `192.168.1.20/24` with a resolvable hostname (e.g., `ipa.example.test`).
- Proper forward and reverse DNS records for the hostname and IP address, or the ability to let FreeIPA manage DNS.
- Access to time synchronization (e.g., `chronyd`) so Kerberos tickets remain valid.

Verify that the network interface uses the expected address:

```bash
ip address show
```

Ensure that `/etc/hosts` contains the host's FQDN and IP if DNS is not yet in place:

```bash
echo "192.168.1.20 ipa.example.test ipa" | sudo tee -a /etc/hosts
```

## System Preparation

### Update the System

Keeping the system current avoids mismatched dependencies.

```bash
sudo dnf update -y
sudo reboot
```

After the reboot, reconnect and confirm time synchronization:

```bash
sudo systemctl status chronyd --no-pager
```

### Configure Firewall

FreeIPA requires access to multiple services, including Kerberos (TCP/UDP 88), LDAP (TCP/UDP 389), and HTTP/HTTPS.

```bash
sudo firewall-cmd --permanent --add-service=freeipa-ldap
sudo firewall-cmd --permanent --add-service=freeipa-ldaps
sudo firewall-cmd --permanent --add-service=freeipa-replication
sudo firewall-cmd --permanent --add-service=kerberos
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --permanent --add-service=ntp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

If DNS will be managed elsewhere, you can omit the `dns` service entry.

### Set the Hostname

```bash
sudo hostnamectl set-hostname ipa.example.test
```

Verify the hostname maps to the static IP:

```bash
getent hosts ipa.example.test
```

## Install FreeIPA Server Packages

Install the required packages from AlmaLinux repositories.

```bash
sudo dnf install -y freeipa-server freeipa-server-dns
```

The `freeipa-server-dns` package is optional; include it if you want FreeIPA to manage DNS records.

## Run the FreeIPA Installer

The `ipa-server-install` utility configures Kerberos, LDAP, DNS, and certificate services.

```bash
sudo ipa-server-install \
  --hostname=ipa.example.test \
  --ip-address=192.168.1.20 \
  --realm=EXAMPLE.TEST \
  --domain=example.test \
  --ds-password='StrongDirectoryPassw0rd!' \
  --admin-password='StrongAdminPassw0rd!' \
  --setup-dns \
  --forwarder=8.8.8.8
```

During the interactive prompts:

1. Confirm the host information and realm/domain settings.
2. Choose to configure DNS if you included `--setup-dns`.
3. Provide DNS forwarders or allow FreeIPA to act as the primary resolver.
4. Accept the default certificate subject base or customize as needed.

The installer configures services, generates certificates, and enables `httpd`, `krb5kdc`, and related daemons.

## Post-Installation Tasks

### Enable the Web UI

FreeIPA provides an admin portal at `https://ipa.example.test/ipa/ui`. Import the CA certificate into your browser if prompted.

### Create Administrative Users

```bash
kinit admin
ipa user-add devops --first=Dev --last=Ops --shell=/bin/bash --password
```

### Enroll Client Hosts

From a client machine joined to the same network, install the client package and enroll it:

```bash
sudo dnf install -y freeipa-client
sudo ipa-client-install --server=ipa.example.test --domain=example.test --realm=EXAMPLE.TEST
```

### Configure DNS Records

If FreeIPA manages DNS, add host records as needed:

```bash
ipa dnsrecord-add example.test app01 --a-rec=192.168.1.30
```

### Verify Services

Check the status of critical services to ensure the deployment is healthy:

```bash
sudo ipa-healthcheck --failures-only
sudo systemctl status ipa --no-pager
```

## Backup and Maintenance

- Schedule regular backups using the `ipa-backup` tool.
- Monitor Kerberos ticket expiration with `klist` and renew as needed.
- Apply updates periodically to benefit from security fixes.

```bash
sudo ipa-backup --online
```

Store the generated backup archives in an off-host location.

## Conclusion

With FreeIPA installed on AlmaLinux and configured for the `192.168.1.20/24` network, you now have a centralized identity platform. Use the web UI and CLI utilities to manage users, hosts, and policies consistently across your environment.

## View This Article

Once published, this article will be available at:

```
https://green-to-code-blog.example.com/articles/2025-05-05-installing-freeipa-on-alma-linux.html
```

Replace the domain with your actual site URL if different.
