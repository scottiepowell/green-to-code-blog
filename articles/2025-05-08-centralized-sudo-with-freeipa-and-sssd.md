---
layout: post
title: "Centralizing Sudo with FreeIPA and SSSD"
date: 2025-05-08
author: "Auto Writer"
tags: [FreeIPA, SSSD, Sudo, Identity Management]
nav_order: 5
parent: The IPA Series
grand_parent: Articles
---

# Centralizing Sudo with FreeIPA and SSSD

> Configure FreeIPA sudo rules, coexist peacefully with local `/etc/sudoers`, and keep cached sudo access working even when a client is offline.

## Objectives

- Manage sudo rules centrally in FreeIPA and deliver them to Linux clients with SSSD.
- Demonstrate how centrally managed rules coexist with minimal local `/etc/sudoers` policy.
- Validate that cached sudo rules continue to function during temporary network outages.

## Environment Overview

- FreeIPA domain with groups `ops` and `dev` already defined.
- Clients enrolled with `ipa-client-install` (or equivalent) so that `sssd` is configured with `sudo_provider = ipa`.
- Sudo 1.8 or later installed on clients with the SSSD sudoers plugin enabled.

## Keep Local Sudoers Minimal (But Present)

Local sudo files are still processed before FreeIPA rules, so keep them tidy:

```bash
# /etc/sudoers
Defaults use_pty
@includedir /etc/sudoers.d
```

Avoid blanket entries such as `%wheel ALL=(ALL) ALL` unless they are intentional. Local drop-ins under `/etc/sudoers.d/` remain available for emergency access, but strive to centralize routine policy.

## Register Common Administrative Commands

Add sudo command aliases in IPA so they can be referenced by multiple rules:

```bash
ipa sudocmd-add /usr/bin/systemctl
ipa sudocmd-add /usr/bin/journalctl
ipa sudocmd-add /usr/bin/dnf
ipa sudocmdgroup-add ops-core
ipa sudocmdgroup-add-member ops-core --sudocmds="/usr/bin/systemctl,/usr/bin/journalctl,/usr/bin/dnf"
```

Grouping related binaries keeps policy readable and avoids duplication when new host groups are added later.

## Build the Sudo Rules

### Ops: Full Root Everywhere

Provide the `ops` group unrestricted access across the estate:

```bash
ipa sudorule-add ops-all-root --hostcat=all --runasusercat=all --runasgroupcat=all --cmdcat=all
ipa sudorule-add-user ops-all-root --groups=ops
ipa sudorule-enable ops-all-root
```

### Dev: Limited Commands on Development Hosts

Constrain the `dev` group to a small set of commands on systems in the `hg-dev` host group:

```bash
ipa sudorule-add dev-limited --hostgroups=hg-dev --runasusercat=all
ipa sudorule-add-allow-command dev-limited --sudocmdgroups=ops-core
ipa sudorule-add-user dev-limited --groups=dev
ipa sudorule-enable dev-limited
```

These commands assume the host group `hg-dev` already maps to your development fleet.

## Refresh Client Caches

After changing policy in IPA, nudge the client to pull the updates and confirm visibility:

```bash
sudo sss_cache -E
sudo -l
```

You should see both global rules (`ops-all-root`) and any host-specific restrictions (`dev-limited`) depending on group membership.

## Verify Day-to-Day Behavior

### Limited Dev Access

While logged into a development host as a `dev` user:

```bash
sudo -l
sudo systemctl status example.service
sudo su -
```

The first two commands should succeed (the second prompts for credentials), while `sudo su -` is denied.

### Precedence with Local Overrides

Create a targeted local rule to illustrate coexistence:

```bash
echo '%emergency ALL=(root) NOPASSWD: /usr/bin/id' | sudo tee /etc/sudoers.d/99-local-test
sudo visudo -c
sudo -l
```

The local `/etc/sudoers.d/99-local-test` entry appears alongside IPA-delivered policy, proving that local files are still honored. Keep such overrides narrow and audited.

## Offline Operation Test

Temporarily block directory services (for example, disconnect the network cable, shut down VPN, or firewall off LDAP/Kerberos ports) and test again:

```bash
sudo -l
sudo systemctl status example.service
```

Because SSSD caches sudo rules, allowed commands continue to function for the cache lifetime. If `sudo -l` returns nothing, inspect `/var/log/secure` (or `journalctl -u sssd`) to ensure the sudo responder is running and that clocks are in sync.

## Troubleshooting and Pitfalls

- **SSSD sudo responder disabled:** Confirm `/etc/sssd/sssd.conf` contains `services = nss, pam, sudo` and restart SSSD if the sudo rules never appear.
- **Clock skew:** Even with cached sudo rules, Kerberos still enforces time skew limits for authentication. Keep NTP synchronized.
- **Empty sudo -l output:** Review `/var/log/secure` or `journalctl -u sudo` for plugin errors. Fallback to local `/etc/sudoers` entries if needed.

## Further Reading

- [Red Hat: Centralized configuration of sudo rules using SSSD](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_authentication_and_authorization_in_rhel/centralized-configuration-of-sudo-rules-using-sssd_configuring-authentication-and-authorization-in-rhel)
- [FreeIPA Project: Sudo Integration Overview](https://www.freeipa.org/page/Sudo)

## View This Article

Once published, access the guide at:

```
https://green-to-code-blog.roadmaps.link/articles/2025-05-08-centralized-sudo-with-freeipa-and-sssd.html
```

Bookmark the link for future reference when managing your FreeIPA sudo policy.
