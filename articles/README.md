# Articles Overview

## Ephemeral Pipeline Series Overview
This landing page introduces the three-part Buildozer pipeline and explains how the guide combines Terraform-provisioned ephemeral EC2 instances, containerized Android builds, and automated teardown steps so you can follow the entire workflow in sequence. [Read the overview](https://green-to-code-blog.roadmaps.link/articles/ephemeral-pipeline/).

## The IPA Series Overview
Serving as a hub for FreeIPA coverage, the series page outlines the authentication-focused articles and positions them as hands-on references for Kerberos, SSSD, and identity management tasks in real deployments. [Read the overview](https://green-to-code-blog.roadmaps.link/articles/the-ipa/).

## Ephemeral Pipeline: Python to Android with GitHub Actions and AWS EC2 – Part 1
Launching the series, this post explains how to spin up an ephemeral AWS EC2 environment with Terraform, orchestrated by GitHub Actions, to provision the build infrastructure for compiling Python apps into Android APKs while capturing state for subsequent stages. [Read the article](https://green-to-code-blog.roadmaps.link/articles/2025-05-01-python_to_APK_part1.html).

## Ephemeral Pipeline: Configuring EC2 for Android Builds with Docker and Buildozer – Part 2
Part two walks through configuring the temporary EC2 host with Docker and a Buildozer-ready image, transferring project sources, running containerized builds, and returning the generated APK to GitHub Actions for distribution. [Read the article](https://green-to-code-blog.roadmaps.link/articles/2025-05-01-python_to_APK_part2.html).

## Ephemeral Pipeline: Automated Teardown and Cost Optimization – Part 3
The concluding entry focuses on restoring Terraform state and automating `terraform destroy` inside GitHub Actions so the ephemeral infrastructure is dismantled after builds, cutting cloud costs and tightening security. [Read the article](https://green-to-code-blog.roadmaps.link/articles/2025-05-01-python_to_APK_part3.html).

## How to Install a FreeIPA Server on AlmaLinux
This guide prepares an AlmaLinux host, opens required firewall services, runs the FreeIPA installer, and covers post-install tasks such as web UI access, client enrollment, DNS management, and backups to establish centralized identity services. [Read the article](https://green-to-code-blog.roadmaps.link/articles/2025-05-05-installing-freeipa-on-alma-linux.html).

## Locking Down FreeIPA Firewall Rules on AlmaLinux
To reduce attack surface, this article details the minimal firewalld services and explicit ports a standalone FreeIPA server should expose, along with validation steps, client testing, and hardening tips for ongoing maintenance. [Read the article](https://green-to-code-blog.roadmaps.link/articles/2025-05-06-freeipa-firewall-hardening.html).

## From Allow-All to Least Privilege: FreeIPA HBAC Step-by-Step
Replacing the permissive default HBAC rule, the walkthrough creates role-based user and host groups, builds SSH service policies, validates access with hbactest and live logins, and shares troubleshooting practices for least-privilege enforcement. [Read the article](https://green-to-code-blog.roadmaps.link/articles/2025-05-07-freeipa-hbac-least-privilege.html).

## Centralizing Sudo with FreeIPA and SSSD
Here you learn to manage sudo command groups and rules centrally in FreeIPA, keep local sudoers minimal, confirm caching behavior for offline clients, and troubleshoot SSSD integration so role-specific privileges stay consistent. [Read the article](https://green-to-code-blog.roadmaps.link/articles/2025-05-08-centralized-sudo-with-freeipa-and-sssd.html).

## Hardening Password and Lockout Policies in FreeIPA
This post tightens the global realm password policy, adds stricter overrides for sensitive groups, demonstrates lockout testing, and flags common pitfalls so administrators can prove compliance and security posture improvements. [Read the article](https://green-to-code-blog.roadmaps.link/articles/2025-05-09-freeipa-password-lockout-policies.html).

## Orchestrating FreeIPA User Lifecycles and Group Structures
Covering joiner, mover, and leaver workflows, the article uses staged users, nested groups, preserved accounts, and service principals to keep POSIX attributes and automation-friendly credentials orderly throughout the user lifecycle. [Read the article](https://green-to-code-blog.roadmaps.link/articles/2025-05-10-freeipa-user-lifecycle-and-groups.html).

## Kerberos 101 in FreeIPA
A quick-reference primer on Kerberos fundamentals, this piece reviews ticket concepts, a minimal `krb5.conf`, chrony setup, essential CLI commands, and common troubleshooting patterns that underpin FreeIPA single sign-on. [Read the article](https://green-to-code-blog.roadmaps.link/articles/2025-05-11-kerberos-101-in-freeipa.html).
