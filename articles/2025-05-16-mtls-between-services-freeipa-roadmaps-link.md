---
title: "Mutual TLS (mTLS) Between Services in the ROADMAPS.LINK Environment"
date: 2025-05-16
layout: article
parent: The IPA Series
grand_parent: Articles
summary: "Issue, deploy, and rotate FreeIPA certificates to enforce mutual TLS across ROADMAPS.LINK microservices, message brokers, and APIs."
tags:
  - mtls
  - tls
  - freeipa
  - certificates
  - roadmaps-link
---

## Why Mutual TLS Matters for ROADMAPS.LINK

Roadmaps.link services already trust the FreeIPA certificate authority for SSH, web, and SSO workloads. Extending that trust boundary to inter-service traffic solves three chronic problems at once:

- **Bidirectional authentication:** Every client and server presents a FreeIPA-issued identity before any payload is exchanged.
- **Encrypted, least-privilege channels:** Only holders of still-valid certificates can establish a TLS tunnel, preventing rogue workloads from joining the mesh.
- **Audit-ready service identities:** Because each certificate is bound to a FreeIPA principal, logs and SIEM pipelines can tie connections back to specific microservices instead of opaque IP addresses.

This makes mTLS a natural continuation of the IPA program: replace ad-hoc, manually generated certificates with centrally governed identities.

## Workflow: Issuing Client and Server Certificates

FreeIPA’s `certmonger` integration issues and tracks certificates for both sides of the TLS handshake. The flow below requests a server certificate for the `api01` endpoint and a client certificate for the `svc-worker` microservice:

```bash
# Server certificate for api01.roadmaps.link
sudo ipa-getcert request \
  -d /etc/pki/tls/server/ \
  -n "Server-Cert api01.roadmaps.link" \
  -N CN=api01.roadmaps.link \
  -D api01.roadmaps.link \
  -T serverCert

# Client certificate for microservice svc-worker.roadmaps.link
sudo ipa-getcert request \
  -d /etc/pki/tls/client/ \
  -n "Client-Cert svc-worker.roadmaps.link" \
  -N CN=svc-worker.roadmaps.link \
  -D svc-worker.roadmaps.link \
  -T clientCert
```

Key practices while issuing certificates:

1. **Standardize names.** Adopt a pattern such as `svc-<name>.roadmaps.link` so certificates map cleanly to workloads and automation scripts.
2. **Segregate storage.** Keep server keys under `/etc/pki/tls/server/` and client keys under `/etc/pki/tls/client/`, owned by the service account, mode `600`.
3. **Leverage profiles.** The `serverCert` and `clientCert` FreeIPA profiles can enforce different key usages and lifetimes. Short-lived client certificates reduce blast radius if a workload is compromised.

## Enforcing mTLS at the Reverse Proxy

With certificates enrolled, configure the fronting proxy (Nginx shown) to request a client certificate from every upstream caller. The server block below terminates TLS, validates the client certificate against the FreeIPA CA, and forwards only verified requests to the backend application:

```nginx
server {
  listen 443 ssl;
  server_name api01.roadmaps.link;

  ssl_certificate       /etc/pki/tls/server/fullchain.pem;
  ssl_certificate_key   /etc/pki/tls/server/privkey.pem;
  ssl_client_certificate /etc/pki/tls/ca.pem;
  ssl_verify_client     on;

  location / {
    proxy_pass http://backend_service;
  }
}
```

A few authorization tips for ROADMAPS.LINK:

- Use the certificate’s Common Name (CN) or Subject Alternative Name (SAN) inside Nginx `map` directives to route only approved clients to specific upstreams.
- Combine mTLS with application-level RBAC. Even when TLS allows a connection, the backend should validate the FreeIPA identity before honoring requests.
- Mirror verification settings on message brokers, databases, and internal APIs so the same trust posture applies across the stack.

## Monitoring Expiry and Break-Glass Procedures

Certificates are only as strong as their renewal discipline. Bake the following checks into your operations runbook:

1. **Track active requests.** `sudo getcert list` shows every certmonger-tracked certificate, its status, and expiry date.
2. **Rotate before expiry.** Automate renewal workflows so services request replacements well ahead of deadline:

    ```bash
    sudo ipa-getcert request \
      -d /etc/pki/tls/client/ \
      -n "Client-Cert svc-worker.roadmaps.link" \
      -N CN=svc-worker.roadmaps.link \
      -D svc-worker.roadmaps.link \
      -r
    ```

3. **Plan for emergencies.** Keep a minimally privileged, longer-lived certificate on standby. Use it to keep critical services online while the primary identity is reissued.

Document these procedures with change-control notes so audits can confirm the rotation cadence.

## Operational Best Practices

- **Enforce mTLS on critical paths.** Databases, message queues, and inter-service APIs should all require authenticated clients before establishing sessions.
- **Automate observation.** Emit certificate CNs and serial numbers into application logs so SIEM dashboards can spot unusual clients or expired credentials.
- **Integrate with FreeIPA policies.** Combine Host-Based Access Control (HBAC) with certificate profiles to restrict which hosts and principals may request client versus server certificates.
- **Coordinate with ACME.** Where workloads autoscale, rely on FreeIPA’s ACME interface to mint and rotate certificates without manual `ipa-getcert` calls.

## Summary and Further Reading

Mutual TLS completes the identity story for ROADMAPS.LINK: the same FreeIPA CA that secures human logins now authenticates machine-to-machine calls. By issuing certificates through certmonger, enforcing validation at the proxy, and rehearsing break-glass renewals, your internal services keep talking—but only to verified peers.

- [Mutual TLS (mTLS) Between Services in the ROADMAPS.LINK Environment](/articles/2025-05-16-mtls-between-services-freeipa-roadmaps-link)
- Continue exploring the [IPA Series](/articles/the-ipa/).

