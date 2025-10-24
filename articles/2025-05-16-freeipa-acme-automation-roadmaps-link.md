---
layout: post
title: "Automating Certificates with FreeIPA ACME"
date: 2025-05-16
author: "Auto Writer"
tags: [FreeIPA, ACME, Certificates, Automation]
nav_order: 3
parent: The IPA Series
grand_parent: Articles
---

# Automating Certificates with FreeIPA ACME

> Enable FreeIPA's ACME responder so dynamic workloads in the roadmaps.link environment can request and renew TLS certificates without manual touch.

## Why ACME Matters for roadmaps.link

- **Supports ephemeral infrastructure:** Containers, short-lived build agents, or lab hosts can enroll for certificates as soon as they come online.
- **Keeps control in-house:** You issue `*.roadmaps.link` certificates from the FreeIPA CA instead of depending on public providers.
- **Matches automation tooling:** Popular ACME clients such as Certbot or `acme.sh` plug into your existing pipelines, and FreeIPA's DNS provider API lets them solve DNS-01 challenges programmatically.
- **Reduces operational toil:** Certificate lifecycle management becomes a predictable script instead of a ticket queue.

## Enable the ACME Responder

Run the management utility on an existing FreeIPA CA server inside the roadmaps.link domain:

```bash
sudo ipa-acme-manage enable
sudo ipa-acme-manage status
```

The first command turns on the ACME responder; the second confirms that the responder is enabled and reachable.

## Issue Certificates from Clients

For workloads that can run `acme.sh`, integrate the FreeIPA DNS provider hook so challenges are solved automatically:

```bash
acme.sh --issue --dns dns_ipa -d service1.roadmaps.link --home /path/to/acme
acme.sh --install-cert -d service1.roadmaps.link \
  --key-file /etc/service1/key.pem \
  --fullchain-file /etc/service1/fullchain.pem \
  --reloadcmd "systemctl reload service1"
```

- `dns_ipa` updates the `_acme-challenge` TXT record through FreeIPA's DNS API.
- The installation step stages the certificate and private key where your service expects them and reloads the daemon so the new TLS assets take effect.

## When to Use ACME vs. certmonger

| Scenario | Choose ACME When... | Stick with certmonger When... |
| --- | --- | --- |
| Dynamic micro-services | Instances spin up and down frequently; automation must be self-service. | Certificates are long-lived and centrally administered. |
| GitOps-driven clusters | CI/CD or GitOps workflows already call ACME clients. | You rely on host enrollment profiles and want FreeIPA UI-driven renewals. |
| Internal-only endpoints | Services never need a public CA signature. | External clients require browser-trusted public certificates. |

Combine the approaches: keep certmonger for stable web, proxy, or IdM components, and use ACME where certificate churn is high.

## Operational Tips

- **Implement request pruning:** Track ACME-issued objects so you can remove stale certificate records for decommissioned services.
- **Harden DNS hooks:** Store the FreeIPA API credentials used by `dns_ipa` with least privilege and rotate them regularly.
- **Monitor issuance volume:** Alert on abnormal spikes to detect compromised automation or misconfigured loops.

---

[Read "Automating Certificates with FreeIPA ACME"](/articles/2025-05-16-freeipa-acme-automation-roadmaps-link/)
