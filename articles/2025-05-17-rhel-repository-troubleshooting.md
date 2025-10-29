---
layout: post
title: "Lab Exercise — Repository Troubleshooting on RHEL"
date: 2025-05-17
author: "Auto Writer"
tags: [RHEL, DNF, Troubleshooting]
nav_order: 1
parent: RHEL System Admin Labs
grand_parent: Articles
---

# Lab Exercise — Repository Troubleshooting on RHEL

> Kick off the **RHEL System Admin Labs** series by tracking down and resolving a failed repository that blocks `dnf` updates.

## Scenario

Your RHEL 9 or RHEL 10 system fails to download metadata for one of its repositories because the SSL certificate chain is not trusted. The goal of this lab is to diagnose which repository is broken, apply a tactical fix so updates work again, and then restore secure defaults.

## Learning Objectives

- Practice interpreting `dnf` cache errors and mapping them back to repository definitions.
- Modify repository configuration safely and understand the impact of `sslverify`.
- Restore proper certificate trust so production systems remain secure.

## Prerequisites

- A RHEL 9 or RHEL 10 system with sudo privileges.
- Shell access to edit files under `/etc/yum.repos.d/`.
- A text editor (`vi`, `nano`, or `sed`) you are comfortable using.

## Step 1 — Reproduce the Failure

Start with a clean cache so the failing repository attempts a fresh metadata download.

```bash
sudo dnf clean all
sudo dnf makecache
```

Watch for errors similar to:

```
Error: Failed to download metadata for repo 'crb':
  - Curl error (60): SSL peer certificate or SSH remote key was not OK for https://…
```

Take note of the repository ID in the error message (`crb` in this example). You will use it to locate the misconfigured file.

## Step 2 — Inspect Enabled Repositories

List every enabled repository along with the file that defined it.

```bash
sudo dnf repolist -v
ls /etc/yum.repos.d/
```

`dnf repolist -v` shows a table that includes `Repo-id`, `Repo-name`, and the `Repo-baseurl` or `Repo-mirrors`. Match the failing repository ID to the filename in `/etc/yum.repos.d/`—for example, `rhel-crb.repo` or `almalinux-crb.repo`.

## Step 3 — Apply a Tactical Fix

Open the file that defines the failing repository and disable certificate verification temporarily by setting `sslverify=0` inside its section.

```bash
sudo vi /etc/yum.repos.d/rhel-crb.repo
```

Add or modify the repository stanza so it resembles:

```
[crb]
name=RHEL $releasever - CRB
baseurl=https://cdn.redhat.com/content/dist/rhel9/$basearch/crb/os
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
metadata_expire=86400
sslverify=0
```

Save the file when finished.

## Step 4 — Validate Metadata Download

Flush cached metadata again, then rebuild the cache to confirm the repository succeeds.

```bash
sudo dnf clean all
sudo dnf makecache
```

This time the command should complete without SSL or metadata errors. If the failure persists, double-check that you edited the correct `.repo` file and that the repository ID matches the error output.

## Step 5 — Restore Secure Defaults (Recommended)

Running with `sslverify=0` suppresses TLS validation and should only be used as a stopgap. Once updates work, fix the underlying certificate trust.

1. Re-enable verification by editing the same `.repo` file and changing `sslverify=0` back to `sslverify=1` (or removing the line).
2. Ensure the root certificate bundle is current:

   ```bash
   sudo dnf install -y ca-certificates
   sudo update-ca-trust extract
   ```

3. If your organization uses custom CAs, place them under `/etc/pki/ca-trust/source/anchors/` before running `update-ca-trust`.

## Step 6 — Confirm End-to-End Success

With trust restored, run a full update to ensure every repository responds correctly.

```bash
sudo dnf update -y
```

The command should finish without repository errors. Investigate further if any repository still fails—often the base URL or proxy settings need attention.

## Verification Checklist

- [ ] `dnf makecache` completes without SSL errors.
- [ ] The problematic repository loads metadata successfully after editing the `.repo` file.
- [ ] `sslverify` is re-enabled once CA trust is repaired.
- [ ] `dnf update -y` runs cleanly.

## Wrap-up

This lab demonstrates a safe way to unblock updates when a repository breaks, while reinforcing the importance of restoring secure TLS validation once the root cause is addressed. Future entries in the **RHEL System Admin Labs** series will tackle other real-world troubleshooting scenarios.

👉 [Lab Exercise — Repository Troubleshooting on RHEL](/articles/2025-05-17-rhel-repository-troubleshooting)
