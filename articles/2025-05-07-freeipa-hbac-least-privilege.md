---
layout: post
title: "From Allow-All to Least Privilege: FreeIPA HBAC Step-by-Step"
date: 2025-05-07
author: "Auto Writer"
tags: [FreeIPA, HBAC, Security, SSH]
nav_order: 1
parent: Articles
---

# From Allow-All to Least Privilege: FreeIPA HBAC Step-by-Step

> A practical walkthrough for replacing FreeIPA's permissive default host-based access control (HBAC) rule with role-driven SSH access across dev, stage, and prod hosts.

## Why Replace the Default HBAC Posture?

FreeIPA ships with an **Allow all** HBAC rule so freshly enrolled clients can contact the identity services without friction. That convenience comes at a cost: every enrolled user gains shell access to every enrolled host. Moving to least-privilege HBAC ensures only the right people can log in to the right systems, while read-only accounts retain IPA access for APIs or the web UI without a shell.

This guide demonstrates how to:

- Model three user roles (`ops`, `dev`, and `readonly`).
- Categorize infrastructure into host groups (`hg-dev`, `hg-stage`, `hg-prod`).
- Scope SSH access with HBAC service groups and rules.
- Prove the policy with `hbactest` dry runs and real SSH attempts.

## Lab Prerequisites

- A functioning FreeIPA domain with administrative privileges (`ipa` CLI or Web UI).
- At least two enrolled hosts, e.g. `dev01.roadmaps.link` and `prod01.roadmaps.link`.
- Optional staging hosts can be added later, but we create the host group up front.
- IPA-enrolled user accounts for each persona:
  - `alice` (developer)
  - `bob` (operations)
  - `carol` (read-only)

All commands below run from the FreeIPA server as `admin` unless noted otherwise.

## Step 1 – Create Role & Host Groups

Start by defining the three user groups that represent your access tiers:

```bash
ipa group-add ops --desc="Operations"
ipa group-add dev --desc="Developers"
ipa group-add readonly --desc="Read-only users"
```

Next, define host groups that segment the environment by lifecycle:

```bash
ipa hostgroup-add hg-dev --desc="Dev hosts"
ipa hostgroup-add hg-stage --desc="Stage hosts"
ipa hostgroup-add hg-prod --desc="Prod hosts"
```

Attach enrolled hosts to their respective groups. For the example inventory:

```bash
ipa hostgroup-add-member hg-dev --hosts=dev01.roadmaps.link
ipa hostgroup-add-member hg-prod --hosts=prod01.roadmaps.link
```

> Add staging hosts with additional `ipa hostgroup-add-member` calls whenever they become available.

## Step 2 – Build an HBAC Service Group for SSH

HBAC rules reference *service* entries. Define a dedicated service group so every SSH-related rule stays consistent:

```bash
ipa hbacsvc-add sshd
ipa hbacsvcgroup-add hbac-svc-ssh
ipa hbacsvcgroup-add-member hbac-svc-ssh --hbacsvcs=sshd
```

You can reuse `hbac-svc-ssh` in future rules (sudo, login shells, etc.) to keep service membership centralized.

## Step 3 – Retire the Broad Default Rule

Confirm the stock `allow_all` rule still exists:

```bash
ipa hbacrule-find
```

Either disable it outright or scope it to administrators only. Disabling is the quickest path to least privilege:

```bash
ipa hbacrule-disable allow_all
```

If you prefer to retain a break-glass option, edit the rule so only trusted admin groups remain in its user list.

## Step 4 – Author Role-Based HBAC Rules

With the groundwork finished, write explicit HBAC rules that describe who may SSH to which hosts.

### 4.1 Operations: SSH to Any Host

Operations staff keep full shell access everywhere. Assign the `ops` group to a rule that covers every enrolled host:

```bash
ipa hbacrule-add ops-ssh-all --servicecat=all
ipa hbacrule-add-user ops-ssh-all --groups=ops
ipa hbacrule-add-host ops-ssh-all --hostcat=all
ipa hbacrule-mod ops-ssh-all --hbacsvcgroups=hbac-svc-ssh
ipa hbacrule-enable ops-ssh-all
```

The `--hostcat=all` flag future-proofs the rule for new hosts without additional maintenance.

### 4.2 Developers: SSH to Dev & Stage Only

Developers should never touch production. Limit the `dev` group to the development and staging host groups:

```bash
ipa hbacrule-add dev-ssh-devstage
ipa hbacrule-add-user dev-ssh-devstage --groups=dev
ipa hbacrule-add-host dev-ssh-devstage --hostgroups=hg-dev --hostgroups=hg-stage
ipa hbacrule-mod dev-ssh-devstage --hbacsvcgroups=hbac-svc-ssh
ipa hbacrule-enable dev-ssh-devstage
```

Host membership changes to `hg-dev` or `hg-stage` immediately affect developer reach without editing the rule.

### 4.3 Read-Only Accounts: No Shell

Read-only users should authenticate to APIs or the Web UI but never receive an interactive shell. Set their IPA login shell to `/sbin/nologin` (or `/usr/sbin/nologin` depending on distro):

```bash
ipa user-mod carol --shell=/sbin/nologin
```

Because HBAC rules grant access, there is no need for an explicit deny rule. Even if a readonly user matches a rule unintentionally, `sshd` refuses the session when `sshd_config` honors the `nologin` shell.

## Step 5 – Validate with `hbactest`

Before touching clients, simulate decisions directly on the IPA server:

```bash
hbactest --user alice --servicename sshd --hostname dev01.roadmaps.link
hbactest --user alice --servicename sshd --hostname prod01.roadmaps.link
hbactest --user bob   --servicename sshd --hostname prod01.roadmaps.link
hbactest --user carol --servicename sshd --hostname dev01.roadmaps.link
```

Expected outcomes:

| Scenario | Result |
| --- | --- |
| `alice` → `dev01` | **allowed** (developer on dev host) |
| `alice` → `prod01` | **denied** (developer blocked from prod) |
| `bob` → `prod01` | **allowed** (ops rule matches all hosts) |
| `carol` → `dev01` | **denied** (shell is `/sbin/nologin`) |

Investigate unexpected denials for time skew, DNS resolution, or group membership typos.

## Step 6 – Confirm with Real SSH Logins

Once the dry run looks correct, hop onto client machines and test interactive logins:

```bash
ssh alice@dev01.roadmaps.link   # succeeds
ssh alice@prod01.roadmaps.link  # permission denied
ssh bob@prod01.roadmaps.link    # succeeds
ssh carol@dev01.roadmaps.link   # disconnected (nologin)
```

Review `/var/log/secure` (RHEL-based) or `/var/log/auth.log` (Debian-based) if any attempts behave differently than `hbactest` predicted.

## Troubleshooting & Maintenance Tips

- **SSSD cache:** Clients cache HBAC data. After policy edits, clear the cache (`sudo sss_cache -E`) or wait for the default refresh window.
- **Time & DNS:** Kerberos tickets fail if clocks drift or SRV records are missing. Verify `klist` and DNS lookups before blaming HBAC.
- **Audit artifacts:** Capture screenshots of HBAC rule summaries, detail pages, and the `hbac-svc-ssh` membership view for change records.
- **Automation:** Wrap the CLI steps in Ansible, Terraform, or shell scripts to repeat the configuration across environments.

## Conclusion

By disabling the blanket `allow_all` rule, introducing environment-aware host groups, and granting access strictly by role, you ensure SSH access in FreeIPA reflects business intent. Developers stay productive on dev and stage, operations retain production reach, and read-only accounts authenticate safely without shell access—all verifiable with `hbactest` and real SSH sessions.

## View This Article

After publishing, view the full guide at:

```
https://green-to-code-blog.roadmaps.link/articles/2025-05-07-freeipa-hbac-least-privilege.html
```

Bookmark the URL for future HBAC audits and onboarding refreshers.
