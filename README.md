# 🛡️ Automated Zero Trust Architecture (NIST SP 800-207)

## 📝 Project Overview
This repository implements a full-stack **Zero Trust Architecture (ZTA)** on a decentralized corporate environment (TechN) following the core philosophy: **"Never Trust, Always Verify"**. 

The goal is to securely isolate critical assets (MariaDB database, PHP application, and SSH administrative access) from untrusted public and domestic networks without relying on traditional network perimeters.

### 🏗️ Network & Logical Architecture
Following **NIST SP 800-207**, the infrastructure strictly decouples the control plane from the data plane:
* **Control Plane:** Tailscale (Coordination Server) & LemonLDAP::NG acting as the **Policy Decision Point (PDP)**.
* **Data Plane:** WireGuard encrypted tunnels & Nginx acting as the **Policy Enforcement Point (PEP)**.

| Machine Name | ZTA Role | Technical Stack |
| :--- | :--- | :--- |
| **zero-client** | Subject | Remote Worker Workstation |
| **test1** | Policy Enforcement Point (PEP) | Nginx, PHP-FPM, MariaDB |
| **auth** | Policy Decision Point (PDP) | LemonLDAP::NG SSO Server |

---

## 🛠️ Implemented Security Controls

### 1. Software-Defined Perimeter (SDP) & Micro-segmentation
To mitigate **lateral movement** from untrusted LANs, the network layer is completely obfuscated using cryptographically verified identities.
* **Network Isolation:** Deployment of a virtual `tailnet` (WireGuard mesh network). Hostnames resolve exclusively to `100.x.y.z` internal IPs.
* **Default Deny Policy:** Explicit JSON Access Control Lists (ACLs) applied via tags (`tag:web`, `tag:sso`, `tag:client`).
* **Kernel-level Network Lock:** Hardened local interfaces (`enp0sX`) using `iptables` to drop any non-loopback or non-tailscale incoming traffic.

```bash
# Proximity network locking snippet applied on nodes
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -i tailscale0 -j ACCEPT
sudo iptables -P INPUT DROP
```

---

### 2. Identity-Driven Administration (Tailscale SSH)
Replaced traditional static secrets (vulnerable SSH private keys on disk or passwords) with real-time Identity Provider (IdP) authentication.
* **Host Hardening:** Complete disabling of `PubkeyAuthentication` and `PasswordAuthentication` on global network interfaces.
* **SSH Interception:** Delegated SSH access validation to the Tailscale Control Plane.
* **DevOps Resiliency:** Integrated a conditional `Match Address 127.0.0.1` gateway to preserve secure local Vagrant maintenance.

---

### 3. Core Service Hardening & Kernel Sandboxing (Zero-TCP Architecture)
If the Web application layer gets compromised, the system uses deep Defense-in-Depth layers to neutralize system-wide privilege escalation and reverse shells.
* **Zero Internal TCP Stack:** Disabled TCP networking in MariaDB (`skip-networking = 1`). Communication is strictly restricted to secure **Unix Sockets** (`/run/php/appphp-fpm.sock`) with restricted permissions (`0660`).
* **OS-Level Identity Auth:** Swapped traditional db-passwords for native kernel validation (`IDENTIFIED VIA unix_socket`) for the dedicated `appphp` system worker.
* **Systemd Sandboxing Cage:** Applied strict security profiles reducing the PHP-FPM exposure index on the host:
  * `RestrictAddressFamilies=AF_UNIX` (Completely blocks outbound internet requests, **neutralizing Reverse Shells**).
  * `ProtectHome=yes` & `PrivateTmp=yes` (File system isolation).
  * `NoNewPrivileges=yes` (Prevents local SUID root privilege escalations).

---

### 4. Application Gateway & Contextual MFA (Layer 7 PEP)
* **Continuous L7 Verification:** Nginx implements an automated `auth_request /lmauth` callback sub-request. No static or dynamic file is served without explicit active token validation by LemonLDAP::NG.
* **Multi-Factor Authentication:** Enforced **TOTP (Time-based One-Time Password)** registration on the SSO portal combined with contextual endpoint validation.

---

## 🔍 Continuous Auditing & System Sanity
To verify compliance against strict **CIS Benchmarks**, automated audit hooks run continuously:
* **Memory Protection:** Randomization of memory addresses via kernel hardening (`kernel.randomize_va_space = 2`) to counter Buffer Overflow exploits.
* **Rootkit Integrity Scans:** Runtime security posture scanning using **Lynis** (Target Baseline Score achieved: 61) and host binary signature auditing via **chkrootkit**.

---

## 📈 Identified Limits & Mitigations
* **Identity Provider Dependency:** Security relies heavily on the root Identity Provider. *Mitigation: Enforce strict multi-factor conditional access policies at the corporate email/GitHub root level.*
* **Control Plane SPOF:** Relies on Tailscale coordination servers availability. *Mitigation: Evaluate high-availability self-hosted control planes like Headscale for critical infrastructure nodes.*
