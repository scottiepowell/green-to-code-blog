---
layout: post
title: "Host & Service Certificates via certmonger (IPA CA + certmonger)"
date: 2025-05-16
author: "Auto Writer"
tags: [FreeIPA, Certificates, Automation]
nav_order: 12
parent: The IPA Series
grand_parent: Articles
---

# Host & Service Certificates via certmonger (IPA CA + certmonger)

> Automate certificate issuance and renewal for every Roadmaps.link service by pairing FreeIPA's CA with certmonger tracking.

FreeIPA ships with a capable certificate authority, but its real power emerges when you let certmonger (and the IPA wrapper `ipa-getcert`) handle the full lifecycle. In the roadmaps.link environment this combination keeps Apache, Nginx, Keycloak, and other workloads enrolled with trusted TLS certificates that never expire unexpectedly.

## Why certmonger + FreeIPA Matters

- **Eliminate manual CSR handling:** Skip ad-hoc `openssl` commands and the risk of forgetting renewals.
- **Ensure consistent trust:** Every host and service gets certificates anchored to the shared FreeIPA CA.
- **Control SANs with confidence:** Specify Subject Alternative Names (SANs) per service and update them later without downtime.
- **Audit friendly:** Apply shared profiles (for example, `httpdssl`) so lifetimes, key usage, and renewal windows stay uniform.

## Prerequisites

1. The host is already enrolled with FreeIPA (`ipa-client-install`).
2. The FreeIPA CA exposes the desired profiles (such as `httpdssl`, `nginxssl`, or `serverCert`).
3. System packages are up to date and you have sudo access.

## Walk-through: Apache on `web01.roadmaps.link`

Install and enable certmonger on the enrolled host:

```bash
sudo dnf -y install certmonger   # or: sudo apt install certmonger
sudo systemctl enable --now certmonger
```

Request and track an Apache HTTP Server certificate by pointing `ipa-getcert` at the HTTPD NSS database and SAN set:

```bash
sudo ipa-getcert request \
  -d /etc/httpd/alias/ \
  -n "Server-Cert web01.roadmaps.link" \
  -N CN=web01.roadmaps.link \
  -D web01.roadmaps.link \
  -D www.web01.roadmaps.link \
  -T httpdssl
```

certmonger stores the request, obtains the certificate from the FreeIPA CA, and watches it for renewal.

## Monitor Issued Certificates

Use the `getcert` helpers to view tracked certificates or watch for expirations:

```bash
sudo getcert list
sudo getcert list --monitor
```

certmonger renews certificates automatically before the expiry window defined by the profile (for example, 30 days prior to a 2-year lifetime).

## Updating SANs Later

Need to add another DNS name—such as `api.web01.roadmaps.link`—after the first issuance? Re-run `ipa-getcert request` with the expanded SAN list and certmonger updates the tracked certificate in-place:

```bash
sudo ipa-getcert request \
  -d /etc/httpd/alias/ \
  -n "Server-Cert web01.roadmaps.link" \
  -D web01.roadmaps.link \
  -D www.web01.roadmaps.link \
  -D api.web01.roadmaps.link \
  -T httpdssl
```

## Deploying to Other Services

### Nginx on `nginx1.roadmaps.link`

```bash
sudo ipa-getcert request \
  -d /etc/nginx/ssl/ \
  -n "Server-Cert nginx1.roadmaps.link" \
  -N CN=nginx1.roadmaps.link \
  -D nginx1.roadmaps.link \
  -T nginxssl
```

### Keycloak on `keycloak.roadmaps.link`

```bash
sudo ipa-getcert request \
  -d /etc/keycloak/ssl/ \
  -n "Server-Cert keycloak.roadmaps.link" \
  -N CN=keycloak.roadmaps.link \
  -D keycloak.roadmaps.link \
  -T serverCert
```

Each command stores keys under the service-specific directory so the application can load them directly (for example, from `/etc/nginx/ssl/` for Nginx).

## Client Trust and Pinning

Distribute the FreeIPA CA root certificate to every client so browsers, curl, and application SDKs validate the service chains:

```bash
sudo cp /etc/ipa/ca.crt /etc/pki/ca-trust/source/anchors/roadmaps-link-ca.crt
sudo update-ca-trust
```

On Debian/Ubuntu, run `sudo update-ca-certificates` instead of `update-ca-trust`. Once the CA bundle is refreshed, clients trust endpoints such as `https://keycloak.roadmaps.link` automatically.

## Lifecycle Awareness

- certmonger handles host/service certificate renewals without intervention, provided the client remains enrolled and reachable.
- When FreeIPA's own CA certificate approaches expiry, plan a coordinated renewal across the environment.
- Monitor `getcert list` output during maintenance windows to catch unreachable hosts before certificates lapse.

## Best Practices Recap

- Standardize on `ipa-getcert` + certmonger for Roadmaps.link services.
- Maintain profile coverage for web servers, proxies, and application SSO components.
- Track expiry with `getcert list --monitor` and integrate alerts if possible.
- Push the FreeIPA CA to all client trust stores to avoid "untrusted issuer" surprises.
- Retire self-signed or unmanaged certificates to keep compliance and audits simple.

Stay on top of certificate hygiene with FreeIPA and certmonger and you eliminate a whole class of preventable outages.

[Host & Service Certificates via certmonger (IPA CA + certmonger)](./2025-05-16-host-service-certificates-certmonger)
