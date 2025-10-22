---
layout: post
title: "Hardening Password and Lockout Policies in FreeIPA"
date: 2025-05-09
author: "Auto Writer"
tags: [FreeIPA, Security, Password Policy, Identity Management]
nav_order: 6
parent: Articles
---

# Hardening Password and Lockout Policies in FreeIPA

> Enforce strong global password requirements, layer on stricter group-specific rules, and validate lockout behavior end to end.

## Objectives

- Define a global password policy that covers length, complexity, history, and lockout settings.
- Override the default policy for sensitive groups, such as operations, with tighter limits.
- Configure lockout and backoff timers, then validate the experience with real test accounts.

## Environment Setup

- FreeIPA domain with existing `ops` and `dev` groups.
- Test identities: `ops_alice` (member of `ops`) and `dev_bob` (member of `dev`).
- Clients joined with `ipa-client-install`, using SSSD for authentication.

## Build the Policy Baseline

### Set the Global Password Policy

Start by tightening the default realm-wide policy. The following command sets length, history, and lockout attributes in one shot:

```bash
ipa pwpolicy-mod --minlength=14 --history=10 --maxfail=5 --failinterval=300 --lockouttime=900
```

If your distribution does not enforce character classes through FreeIPA alone, layer complexity checks on the client side via PAM (for example `pam_pwquality` on RHEL derivatives or `pam_cracklib` on older Debian-based systems). Document the distro-specific steps you take so other administrators can replicate them.

### Create a Stricter Policy for Operations

Attach a dedicated policy to the `ops` group with higher requirements and a longer lockout window:

```bash
ipa pwpolicy-add ops --minlength=20 --history=15 --maxfail=3 --lockouttime=1800
```

Because FreeIPA maps policy by group, any member of `ops` automatically inherits these tighter controls.

### Optional Expiration and Grace Settings

If you also need rotating passwords, set a maximum lifetime and grace logins globally:

```bash
ipa pwpolicy-mod --maxlife=90 --gracelimit=7
```

Communicate the change to users, especially if they rely on cached credentials while offline.

## Verification and Testing

1. Attempt to change `dev_bob`'s password to something weak. The CLI should reject it and report which server-side rule blocked the change.
2. Trigger a lockout by performing more than `--maxfail` bad authentications for a user. Review `/var/log/dirsrv/*/access` and KDC logs to confirm the lock event, then verify the account unlocks automatically after the configured `--lockouttime` or via administrative reset.
3. Confirm that `ops_alice` receives the stricter policy with:
   ```bash
   ipa pwpolicy-show --user=ops_alice
   ```

## Pitfalls to Watch

- **Client complexity enforcement** – Some clients may fail password changes locally because of PAM requirements before the request reaches FreeIPA. Note which layer rejected the password so troubleshooting remains clear.
- **SSSD caching delays** – Cached credentials can postpone visible password-expiration prompts on clients. Plan for a grace period and communicate expectations to end users.

## Artifacts

Capture both console output and UI evidence for documentation and audits:

- **Screenshots** – Global policy page and the `ops` override in the FreeIPA web UI.
- **CLI transcripts** – Failed password change attempts, lockout notifications, and the unlock workflow.

With these artifacts in hand, you have a repeatable playbook for enforcing and demonstrating compliant password hygiene within FreeIPA.
