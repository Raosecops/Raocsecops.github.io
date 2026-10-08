---
layout: post
title: "Engineering an Enterprise-Grade Zero-Trust Bastion Architecture: Network Segmentation & Host Hardening"
date: 2026-10-08 09:00:00 +0300
categories: [Cloud Security, Network Defense, Enterprise Architecture]
tags: [Zero-Trust, UFW, Fail2Ban, SSH, Bastion, Infrastructure Hardening]
---

When designing secure cloud infrastructure, exposing core production workloads directly to public-facing networks introduces unacceptable attack vectors. Automated botnets continuously scan the global IP space, targeting exposed SSH daemons with relentless brute-force and credential-stuffing campaigns.

As part of a recent client hardening engagement, I was tasked with engineering an isolated, defense-in-depth architecture to eliminate public exposure for internal cloud assets. This post details the implementation of a **two-tier Zero-Trust bastion gateway** designed to enforce strict network segmentation, cryptographic access controls, and active threat monitoring.

---

## 1. Threat Modeling & Architecture Design

To achieve a true Zero-Trust posture, the primary directive was simple: **internal assets must have zero direct routes to or from the public internet.**

The deployed topology divides the infrastructure into two segregated network tiers bridged via the hypervisor fabric:
* **The Bastion Tier (Edge Gateway):** The sole authorized public-facing entry point. It absorbs inbound operator traffic under strict ingress constraints.
* **The Production Tier (`prd-euw`):** An isolated internal server environment possessing no public IP allocation. Interaction with this tier requires routing exclusively through the bastion.

### Streamlining Multi-Hop Access via OpenSSH ProxyJump
To maintain operational efficiency without compromising security or storing raw private keys on intermediate nodes, client operators access the production tier seamlessly using native OpenSSH configurations (`~/.ssh/config`):

text
Host bastion
HostName 10.229.49.x
User ubuntu
IdentityFile ~/.ssh/id_ed25519
ForwardAgent yes

Host prd-euw
    HostName 10.229.xx.xxx
    User ubuntu
    ProxyJump bastion
    IdentityFile ~/.ssh/id_ed25519

When initiating an SSH session to the production node (`ssh prd-euw`), the client transparently constructs an encrypted tunnel through the bastion gateway, establishing a secure end-to-end channel.

---

## 2. Hardening the Attack Surface

Securing the infrastructure required stripping away legacy protocol defaults and enforcing rigorous hardening baselines:

* **Cryptographic Enforcement:** In `/etc/ssh/sshd_config`, fallback password authentication and administrative root logins were entirely prohibited:

PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes

* **Perimeter Defense (`UFW`):** Implemented strict default-deny ingress and egress policies, permitting only necessary administrative pathways.
* **Intrusion Prevention (`Fail2Ban`):** Configured active monitoring jails on the SSH daemon to dynamically ban and drop IPs exhibiting suspicious or brute-force behavior.

---

## 3. Security Validation & Penetration Testing

To verify that the hardening controls met enterprise compliance standards, I executed a controlled validation assessment by simulating an automated credential-stuffing attack against the internal production node from the bastion:

bash
hydra -l ubuntu -p wrongpassword 10.229.xx.xxx ssh -t 4

**Execution Output:**

text
[ERROR] target ssh://10.229.49.xxx:xx/ does not support password authentication (method reply 4).

Because password authentication is disabled at the protocol level, the target host rejects login attempts instantly before any credentials can be evaluated—completely neutralizing credential-stuffing and automated password-spraying attacks.

---

## Conclusion

This engagement delivered a robust, production-ready security architecture that successfully eliminated public exposure for internal client workloads. By integrating strict network micro-segmentation, cryptographic key enforcement, and automated intrusion prevention, the infrastructure achieves a resilient Zero-Trust posture aligned with modern enterprise standards.



