---
layout: post
title: "Lab Exercise — Installing Red Hat Ansible Automation on RHEL"
date: 2025-05-18
author: "Auto Writer"
tags: [Ansible, RHEL, Automation]
nav_order: 1
parent: Red Hat Ansible
grand_parent: Articles
---

# Lab Exercise — Installing Red Hat Ansible Automation on RHEL

> Launch the new **Red Hat Ansible** series by installing Red Hat-supported Ansible components on RHEL and validating the essential day-one commands every admin should know.

## Scenario

You need to provision Ansible on a fresh Red Hat Enterprise Linux 9 server that has access to Red Hat subscription content. After the installation you must confirm the toolchain works by running the most common `ansible` commands.

## Learning Objectives

- Enable the appropriate Red Hat repositories for Ansible automation content.
- Install `ansible-core` along with supporting command-line tooling.
- Verify connectivity to managed nodes with ad-hoc commands and playbook checks.

## Prerequisites

- A RHEL 9 system registered with Red Hat Subscription Management.
- Sudo privileges on the control node.
- At least one managed host reachable over SSH for testing (can be localhost).

## Step 1 — Enable Red Hat Ansible Repositories

Confirm your system can access the latest Ansible builds by enabling the Automation Platform repository.

```bash
sudo subscription-manager repos --enable=ansible-automation-platform-2.4-for-rhel-9-x86_64-rpms
```

Use `subscription-manager repos --list-enabled` to confirm the repository shows up. If you manage systems through Satellite, ensure the corresponding channel is synced and attached instead of using the public CDN command above.

## Step 2 — Install ansible-core and Supporting Packages

Install `ansible-core` along with recommended utilities for inventory parsing and documentation.

```bash
sudo dnf install -y ansible-core ansible-collection-redhat-rhel_system_roles sshpass
```

During the transaction, review the summary to ensure `python3`, `jinja2`, and `yaml` dependencies resolve correctly. If you prefer a lean install, you can omit `sshpass`, but it is helpful for quick credential testing in labs.

## Step 3 — Verify the Ansible Version and Configuration Path

Run the two most common post-install commands to confirm the interpreter and configuration search path.

```bash
ansible --version
ansible-config dump --only-changed
```

`ansible --version` reports the Python interpreter, configuration file (`ansible.cfg`), and collection search path. `ansible-config dump --only-changed` highlights any overrides that Ansible detects so you can confirm it is still using defaults on a fresh install.

## Step 4 — Inspect Built-in Documentation and Collections

Validate that Red Hat content and manual pages are available locally.

```bash
ansible-doc -l | head -n 20
ansible-galaxy collection list | grep -E "(ansible|redhat)"
```

`ansible-doc` provides inline module help, while `ansible-galaxy collection list` confirms the Red Hat System Roles collection installed in Step 2.

## Step 5 — Create an Inventory and Test Connectivity

Build a minimal inventory file and test SSH connectivity using the `ping` module.

```bash
mkdir -p ~/ansible-labs
cat <<'INV' > ~/ansible-labs/inventory.ini
[lab]
localhost ansible_connection=local
INV

ansible -i ~/ansible-labs/inventory.ini lab -m ping
```

The module should return `"ping": "pong"` for `localhost`. If you plan to manage remote hosts, replace `localhost` with their hostnames and ensure SSH keys are in place.

## Step 6 — Validate Playbook Syntax and Dry Run

Create a simple playbook and run the top syntax-checking commands before applying changes.

```bash
cat <<'PLAY' > ~/ansible-labs/sample.yml
---
- name: Verify lab connectivity
  hosts: lab
  gather_facts: false
  tasks:
    - name: Print greeting
      ansible.builtin.debug:
        msg: "Hello from Red Hat Ansible"
PLAY

ansible-playbook -i ~/ansible-labs/inventory.ini sample.yml --syntax-check
ansible-playbook -i ~/ansible-labs/inventory.ini sample.yml --check
```

The syntax check should report `playbook: sample.yml`, and the check mode run should finish without changes. These commands are staples when reviewing playbooks before rolling them into production pipelines.

## Verification Checklist

- [ ] The Ansible Automation Platform repository is enabled and visible in `subscription-manager repos --list-enabled`.
- [ ] `ansible --version` and `ansible-config dump --only-changed` report healthy defaults.
- [ ] `ansible -m ping` succeeds against your inventory.
- [ ] `ansible-playbook --syntax-check` and `--check` both complete without errors.

## Wrap-up

You now have a fully operational Red Hat Ansible control node plus a reference set of top commands to validate installations on future hosts. Subsequent entries in the **Red Hat Ansible** series will dive into inventory design, automation controller integration, and content collection management.

👉 [Lab Exercise — Installing Red Hat Ansible Automation on RHEL](/articles/2025-05-18-install-ansible-on-rhel)
