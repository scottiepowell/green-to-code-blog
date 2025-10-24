---
layout: post
title: "OTP / 2-Factor Authentication (2FA) for SSH in the ROADMAPS.LINK Environment"
date: 2025-05-15
author: "Auto Writer"
tags: [FreeIPA, OTP, 2FA, SSH, ROADMAPS.LINK]
nav_order: 12
parent: The IPA Series
grand_parent: Articles
---

# OTP / 2-Factor Authentication (2FA) for SSH in the ROADMAPS.LINK Environment

> Issue FreeIPA-backed one-time passwords so SSH logins across ROADMAPS.LINK require strong, multi-factor assurance without relying on third-party identity providers.

## Overview

FreeIPA ships with a first-party Time-based One-Time Password (TOTP) service that turns ROADMAPS.LINK into its own two-factor authentication provider. By issuing tokens directly from the identity platform, administrators can require a six-digit code whenever users authenticate with SSH—globally, for specific groups such as `ops`, or for handpicked accounts. The entire flow stays inside RoadMaps.link infrastructure: FreeIPA issues the token, SSSD negotiates the challenge, and PAM verifies the response before `sshd` opens a shell.

## 1. Enable and Assign Tokens

### Create a new TOTP token

Run the following commands from any host enrolled in FreeIPA where you have administrator rights:

```bash
ipa otptoken-add \
  --owner=alice \
  --type=totp \
  --digits=6 \
  --time-step=30 \
  --description="Alice Yubi/Authenticator"
```

FreeIPA returns a token UUID and a shared secret. Store the secret securely or render it as a QR code so the user can import it into FreeOTP, Authy, Google Authenticator, or hardware authenticators that understand TOTP. Use `ipa otptoken-show <token-UUID>` to redisplay the details later.

## 2. Require OTP for Groups or Users

### Enforce OTP-only access for a group

To demand an OTP response (no password fallback) for the `ops` group:

```bash
ipa group-mod ops --user-auth-type=otp
```

### Combine password and OTP (true 2FA)

If you want both a password and a one-time code, declare both authentication types:

```bash
ipa group-mod ops --user-auth-type=otp,password
```

Apply the same rule to a single user when you need tighter controls without moving group membership:

```bash
ipa user-mod alice --user-auth-type=otp,password
```

Any subject bound to `--user-auth-type=otp` must now present a valid TOTP code before PAM allows the SSH session to complete.

## 3. SSH and PAM Settings

### OpenSSH configuration

Ensure SSH is prepared to negotiate keyboard-interactive challenges:

```text
/etc/ssh/sshd_config

UsePAM yes
KbdInteractiveAuthentication yes
# For older OpenSSH releases:
# ChallengeResponseAuthentication yes
```

Reload the SSH daemon after saving the configuration:

```bash
sudo systemctl restart sshd
```

### Validate PAM integration

Confirm that the PAM stack calls into SSSD so FreeIPA can broker the OTP challenge/response exchange:

```bash
sudo grep -n "auth.*pam_sss.so" /etc/pam.d/system-auth 2>/dev/null \
  || sudo grep -n "auth.*pam_sss.so" /etc/pam.d/common-auth
```

No additional SSSD configuration is required—FreeIPA automatically coordinates the OTP flow with PAM.

## 4. User Experience

When an account is configured with `--user-auth-type=otp,password`, SSH prompts for the password first and immediately follows with a six-digit OTP challenge. Accounts configured with `--user-auth-type=otp` see only the OTP prompt.

### Example SSH session

```bash
# Clear any cached credentials before testing
ssh alice@dev01.roadmaps.link
# Prompt sequence:
# Password:  ********
# One-time password (6 digits): 123456
```

Successful responses to both prompts result in a standard SSH login backed by multi-factor assurance.

## 5. Per-Group Enforcement Patterns

Many ROADMAPS.LINK environments blend usability and assurance by splitting policies:

- `ops` group → enforce `otp,password` for privileged production access.
- `dev` group → allow password-only for lower-risk development systems.

Align these rules with Host-Based Access Control (HBAC) policies for precise targeting. For example, bind an `ops-ssh-2fa` HBAC rule to the `hg-prod` host group while letting `hg-dev` and `hg-stage` keep single-factor rules. Production servers then require OTP, while developer systems remain streamlined.

## 6. Recovery and Rotation

Lost or replaced devices should not block access or weaken security. Disable the old token and immediately issue a replacement:

```bash
ipa otptoken-mod <UUID> --setattr=ipatokenDisabled=TRUE
ipa otptoken-add --owner=alice --type=totp --digits=6 --time-step=30 \
  --description="Replacement TOTP Token"
```

Encourage users to enroll multiple tokens (for example, mobile authenticator plus hardware key) to stay resilient against device loss.

## 7. Logging and Troubleshooting

Keep an eye on system logs whenever you roll out or troubleshoot OTP requirements:

```bash
journalctl -u sshd --since -30m
journalctl -u sssd_pam -u sssd --since -30m
```

FreeIPA’s Kerberos (`krb5kdc`) and HTTPD logs also record OTP validation details. Search for messages containing `otp`, `pam`, or `sssd` while diagnosing issues. If SSH never prompts for a code, verify that `KbdInteractiveAuthentication yes` is active and that the user or group policy explicitly includes `otp` or `otp,password` in `--user-auth-type`.

## 8. Operational Guidelines for ROADMAPS.LINK

- **Synchronize time:** OTP validation depends on tight clock alignment. Use `chronyc tracking` to confirm offsets across controllers and clients.
- **Maintain DNS integrity:** Confirm `_kerberos._udp.roadmaps.link` SRV records resolve correctly so hosts can reach the FreeIPA services.
- **Refresh caches:** After policy changes, clear local caches with `sudo sss_cache -E` to pick up new authentication requirements.
- **Protect keytabs:** Keep service keytabs mode `600`, owned by `root` or the relevant service account, and never store them in version control.
- **Layered policy:** Combine HBAC (where logins are allowed), sudo (what commands can run), and OTP (how users authenticate) for a comprehensive control plane.

## Final Thought

Integrating FreeIPA-issued OTP tokens into SSH hardens the ROADMAPS.LINK fleet without outsourcing identity to third parties. Administrators gain centralized control over who must use 2FA, while users benefit from a consistent experience that keeps privileged access both secure and streamlined.
