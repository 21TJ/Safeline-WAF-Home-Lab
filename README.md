# Web Application Firewall Deployment & Attack Simulation Lab
### Building, Attacking, and Defending a Vulnerable Web Application with SafeLine WAF

**Category:** Blue Team / Defensive Security · Home Lab Project

---

## 1. Executive Summary

This project documents the design and deployment of a self-contained cybersecurity home lab used to demonstrate how a **Web Application Firewall (WAF)** protects a vulnerable web application from common attacks. I built a two-machine virtualized environment (Kali Linux as the attacker, Ubuntu Server as the target), deployed **DVWA (Damn Vulnerable Web Application)** behind a full LAMP stack, and layered **SafeLine WAF** in front of it as a reverse proxy security control. I then simulated a real SQL Injection attack from the attacker machine and validated that the WAF detected and blocked the malicious traffic before it reached the application.

The goal was to build hands-on experience with the kind of layered, defense-in-depth architecture used in real production environments — and to practice the analyst skill of reading traffic through each layer of the stack (network → web server → application → database) to understand exactly where and how an attack is stopped.

**Skills demonstrated:** network segmentation, Linux server administration, web server/database configuration, DNS resolution, PKI/SSL certificate management, reverse proxy architecture, WAF policy configuration, SQL Injection testing, log analysis, and security documentation.

---

## 2. Objective

- Deploy a vulnerable web application (DVWA) on a Linux server to act as an attack target.
- Place a Web Application Firewall in front of the application as the sole internet-facing entry point.
- Simulate an SQL Injection attack from an attacker machine (Kali Linux) and confirm the WAF blocks it.
- Configure and test additional WAF security controls: HTTP flood protection, authentication gateway, and custom IP-based deny rules.
- Document the full request lifecycle through every layer of the architecture, as a SOC analyst would need to when investigating an alert.

---

## 3. Lab Architecture

```
                         ┌────────────────────┐
                         │   Kali Linux (VM)   │
                         │   Attacker Machine  │
                         └─────────┬───────────┘
                                   │  HTTP/HTTPS Requests
                                   ▼
                         ┌────────────────────┐
                         │   SafeLine WAF      │  <-- Port 443 (public-facing)
                         │  (Nginx reverse     │
                         │   proxy + ruleset)  │
                         └─────────┬───────────┘
                                   │  Inspected / Filtered Traffic
                                   ▼
                         ┌────────────────────┐
                         │   Apache Web Server │  <-- Port 8080 (internal only)
                         │   Ubuntu Server VM  │
                         └─────────┬───────────┘
                                   │
                                   ▼
                         ┌────────────────────┐
                         │   PHP / DVWA        │
                         └─────────┬───────────┘
                                   │
                                   ▼
                         ┌────────────────────┐
                         │   MySQL Database    │  <-- Port 3306
                         └────────────────────┘
```

**Key architectural decision:** Apache was moved off port 80/443 and onto port 8080, so that SafeLine WAF could occupy the public-facing ports (80/443) and act as the single choke point for all inbound traffic. This mirrors how WAFs are deployed in production — as a reverse proxy in front of the real application server, never bypassable.

| Component | Role | Details |
|---|---|---|
| Kali Linux | Attacker / traffic generator | Bridged network adapter, static-mapped hostname |
| Ubuntu Server 22.04 LTS | Target web server | LAMP stack (Apache, MySQL, PHP) |
| DVWA | Vulnerable application | Used to safely generate real exploit traffic |
| SafeLine WAF | Security control under test | Nginx-based reverse proxy WAF |
| `/etc/hosts` | Local DNS resolution | Simulates domain-based access (`dvwa.local`) instead of raw IPs |
| Self-signed SSL cert | Encryption in transit | Issued via OpenSSL, imported into SafeLine |

---

## 4. Build Process

### 4.1 Environment Setup
- Provisioned two VirtualBox VMs (Kali Linux, Ubuntu Server 22.04 LTS) on a shared **bridged network** so both machines behaved as independent hosts on the same LAN — closer to a real network than NAT.
- Verified Layer 3 connectivity between hosts with `ping` before proceeding.

### 4.2 Target Server Build (Ubuntu)
- Installed and configured a full **LAMP stack** (Apache2, MySQL, PHP, `php-mysql`) manually and deliberately — one component at a time — rather than a single bundled install, in order to understand each layer's specific role (web server vs. application runtime vs. database).
- Deployed **DVWA** from source, set correct file ownership/permissions (`www-data`), and configured its database connection.
- Seeded the database with custom test records to have realistic data to target during exploitation testing.

### 4.3 Network & Name Resolution
- Configured `/etc/hosts` on both machines to resolve `dvwa.local` to the Ubuntu server's IP, simulating enterprise-style domain-based access instead of raw IP addressing.
- (Optional extension) Documented how a dedicated **BIND9 DNS server** would replace static host-file mappings in a larger environment.

### 4.4 Port Segregation
- Reconfigured Apache to listen on **8080** instead of 80/443, freeing the standard web ports for the WAF. This enforces that **all traffic must pass through the WAF** — there is no direct path to the application server from outside the host.

### 4.5 Encryption
- Generated a self-signed X.509 certificate with OpenSSL and imported it into SafeLine so the WAF — not Apache — terminates TLS for inbound connections.

### 4.6 WAF Deployment
- Deployed **SafeLine WAF** via its automated install script on the Ubuntu host.
- Onboarded DVWA as a protected application inside SafeLine:
  - Domain: `www.dvwa.local`
  - Backend (reverse proxy target): `http://<Ubuntu_IP>:8080`
  - Public listener: port 443 only (port 80 removed)
  - SSL certificate attached for HTTPS termination at the WAF

---

## 5. Attack Simulation & Detection

### 5.1 Attack
From the Kali Linux VM, I browsed to `https://dvwa.local`, authenticated to DVWA, and set the application's security level to **Low** to allow classic SQL Injection payloads (e.g. `admin' OR '1'='1`) in the login/search fields.

### 5.2 Detection & Blocking
SafeLine WAF inspected the inbound request, matched it against its SQL Injection ruleset, and **blocked the request before it reached Apache/DVWA**. This was confirmed two ways:
- The attacker received a WAF block/error page instead of a normal application response.
- The **SafeLine WAF logs** showed the request flagged and blocked as a detected SQL Injection attempt, including source IP and matched signature.

This is the core validation of the project: a payload that would have compromised the unprotected application was neutralized purely by the WAF layer, with zero changes to the application code itself.

### 5.3 Full Request Lifecycle (as traced for this test)
```
Attacker (Kali) → SafeLine WAF (443, TLS termination + inspection)
   → [BLOCKED — malicious pattern matched, request dropped]
```
Versus a legitimate request:
```
User (Kali) → SafeLine WAF (443) → Apache (8080) → PHP/DVWA
   → MySQL (3306) → Response returns back through the same path
```

---

## 6. Additional WAF Controls Tested

| Control | Purpose | Result |
|---|---|---|
| **HTTP Flood Defense** | Rate-limit requests per source to mitigate DoS/brute-force behavior | Configured request-per-second thresholds and ban duration; validated with repeated automated requests |
| **Authentication Gateway** | Require credentials at the WAF layer before any traffic reaches the app | Enabled; confirmed the WAF prompted for auth prior to passing traffic to DVWA |
| **Custom Deny Rules (IP blocklisting)** | Manually block a known-malicious source | Added a deny rule matching the Kali VM's IP; confirmed subsequent requests were blocked outright |

---

## 7. Key Takeaways / What This Demonstrates

- **Defense-in-depth in practice:** the WAF acted as a compensating control that stopped a real exploitation technique (SQLi) without needing to patch or rewrite the vulnerable application — directly relevant to how SOC/security teams protect legacy or third-party apps they don't fully control.
- **Traffic-flow analysis:** being able to trace a request end-to-end through WAF → web server → application → database is exactly the skill needed to investigate alerts and determine at which layer an attack was stopped (or wasn't).
- **Reverse proxy security architecture:** understanding why the WAF must be the *only* internet-facing entry point (and why the app port had to be moved off 80/443) reflects real production network design.
- **Hands-on log analysis:** validating detections directly from WAF logs rather than assuming success — a core SOC analyst habit.

---

## 8. Possible Extensions

- Add **OWASP Juice Shop** as a second target to broaden attack-surface coverage.
- Test additional attack classes (XSS, command injection, file inclusion) against the same WAF policy.
- Forward SafeLine WAF logs into a home SIEM (e.g., Wazuh, Splunk Free, ELK) for centralized alerting and dashboarding.
- Integrate an NGFW/IPS in front of the WAF for a full layered perimeter.

---

## 9. Environment / Tools Used

- VirtualBox (virtualization)
- Kali Linux (attacker platform)
- Ubuntu Server 22.04 LTS (target platform)
- Apache2, PHP, MySQL (LAMP stack)
- DVWA (Damn Vulnerable Web Application)
- SafeLine WAF (Nginx-based reverse proxy WAF)
- OpenSSL (self-signed certificate generation)
