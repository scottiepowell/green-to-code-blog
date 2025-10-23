---
layout: post
title: "Kerberos SSH with GSSAPI — FreeIPA Integration on Roadmaps.link"
date: 2025-05-13
author: "Auto Writer"
tags: [FreeIPA, Kerberos, SSH, GSSAPI]
nav_order: 3
parent: The IPA
grand_parent: Articles
---

# Kerberos SSH with GSSAPI — FreeIPA Integration on Roadmaps.link

> Deliver password-less, auditable SSH by letting the ROADMAPS.LINK Kerberos realm prove every session instead of local secrets.

## Why Lean on Kerberos for SSH?

- **Password-less sign-in:** Users authenticate once with `kinit`; every SSH hop inside the realm reuses the TGT-derived service tickets.
- **Single source of identity truth:** FreeIPA issues host and user principals, keytabs, and HBAC rules so operators do not juggle local accounts.
- **Cryptographic assurance:** Tickets are verified against host keytabs (`/etc/krb5.keytab`), eliminating shared secrets and replay-prone passwords.

## Big-Picture Flow

1. A user runs `kinit alice@ROADMAPS.LINK` and receives a Ticket-Granting Ticket (TGT).
2. The SSH client asks the KDC for `host/dev01.roadmaps.link@ROADMAPS.LINK` and receives a service ticket.
3. The client presents the ticket to `dev01.roadmaps.link` during SSH negotiation.
4. `sshd` validates the ticket using the host principal stored in its keytab.
5. Access is granted with no password prompt as long as HBAC allows the session.

## Server-Side Requirements

Every enrolled host already carries a Kerberos identity (`host/<fqdn>.roadmaps.link@ROADMAPS.LINK`) and keytab from `ipa-client-install`. Confirm `sshd` is wired for GSSAPI and PAM:

```text
/etc/ssh/sshd_config

UsePAM yes
GSSAPIAuthentication yes
GSSAPICleanupCredentials yes
# AuthenticationMethods gssapi-with-mic          # Kerberos only
# AuthenticationMethods gssapi-with-mic,password # Kerberos, then fallback
```

Reload `sshd` after editing:

```bash
sudo systemctl restart sshd
```

Keep the host keytab secured (`chmod 600 /etc/krb5.keytab`) so only `root` and `sshd` can read the Kerberos keys.

## Client-Side Expectations

Tell OpenSSH to try Kerberos first. You can set this per user or across the fleet:

```text
~/.ssh/config

Host *.roadmaps.link
  GSSAPIAuthentication yes
  GSSAPIDelegateCredentials yes
```

`GSSAPIDelegateCredentials` forwards the user’s ticket cache for services that need Kerberos on the remote host (NFS, internal web apps, or multi-hop SSH).

## Login Runbook

```bash
# 1. Acquire a TGT
kinit alice@ROADMAPS.LINK

# 2. Confirm ticket lifetime and renewability
klist

# 3. Connect to any host that trusts the realm
ssh alice@dev01.roadmaps.link

# 4. Inspect tickets on the remote system
klist
```

If the user’s ticket is expired or the target cannot reach the KDC, the connection falls back according to your `AuthenticationMethods` policy.

## Controlling Access Paths

- **Kerberos-only bastions:** Uncomment `AuthenticationMethods gssapi-with-mic` to reject passwords entirely.
- **Graceful fallback:** Pair Kerberos with passwords (`gssapi-with-mic,password`) while legacy systems are still migrating.
- **HBAC policies:** Use FreeIPA Host-Based Access Control to express “who can SSH where,” such as developers → dev/stage, operators → everywhere, auditors → read-only bastions.

## Audit and Forensics

Log files reference Kerberos principals, giving you non-repudiation for each session.

```bash
# On the target host
journalctl -u sshd --since -30m

# On the FreeIPA KDC
sudo journalctl -u krb5kdc --since -1h 2>/dev/null || true

# On the client
journalctl -u sssd_pam --since -1h 2>/dev/null || journalctl -u sssd --since -1h
last -a
```

Ticket cache forwarding lets you verify delegated credentials on the remote host with `klist`. Align retention policies between the OS logs and FreeIPA so investigations can correlate events.

## Operational Guardrails

- **Time sync or bust:** Kerberos rejects tickets if clocks differ by more than ~5 minutes. Keep Chrony/NTP aligned across IPA servers, clients, and targets.
- **DNS hygiene:** Validate `_kerberos._udp.roadmaps.link` SRV records and forward/reverse lookups (`dig _kerberos._udp.roadmaps.link SRV +short`).
- **SSSD caches:** After HBAC or policy changes, purge clients with `sudo sss_cache -E` or restart `sssd` to pick up new rules.
- **Keytab lifecycle:** Rotate host keytabs via `ipa-getkeytab` on a schedule and document the procedure alongside TLS key management.

## Troubleshooting Checklist

- `ssh -vvv dev01.roadmaps.link` shows whether the client attempted GSSAPI and why it may have fallen back.
- `klist -ef` confirms ticket encryption types and renewal windows before they surprise you mid-session.
- `ipa hbactest --user alice --host dev01 --service sshd` proves whether FreeIPA policies block the login.
- `getent hosts dev01.roadmaps.link` and `getent passwd alice` validate that SSSD is resolving principals correctly.

## Key Takeaways

Kerberos-backed SSH turns ROADMAPS.LINK into a centrally governed identity mesh: users authenticate once, tickets glide across managed hosts, and every session is backed by FreeIPA’s cryptographic audit trail. The result is cleaner operations, fewer passwords to leak, and faster incident response across the environment.
