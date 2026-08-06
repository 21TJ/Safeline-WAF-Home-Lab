# 🛡️ WAF Home Lab — Blocking SQL Injection with SafeLine WAF

I built a home lab to see, hands-on, how a Web Application Firewall actually stops an attack — not just read about it. This report documents a Kali Linux attacker VM, an Ubuntu server running a deliberately vulnerable web app (DVWA), and **SafeLine WAF** sitting in front of it as the only way in. I ran a real SQL Injection payload at it and confirmed the WAF caught and blocked it before it reached the app or database — then tested a few more of SafeLine's controls (flood defense, auth gateway, IP blocking) on top.

The goal wasn't "install a WAF and take a screenshot." It was to actually understand the full path a request takes — network → WAF → web server → app → database — well enough that if I saw an alert like this on the job, I'd know exactly what layer stopped it and why, instead of just trusting a dashboard.

---

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Installation / Lab Setup](#installation--lab-setup)
- [Usage — Running the Attack Simulation](#usage--running-the-attack-simulation)
- [Results](#results)
- [Additional WAF Controls Tested](#additional-waf-controls-tested)
- [What I Learned](#what-i-learned)
- [Possible Extensions](#possible-extensions)
- [License](#license)
- [Contact](#contact)

---

## Architecture

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

**Key design decision:** Apache was moved off ports 80/443 and onto 8080, so SafeLine WAF could take the public-facing ports and act as the single choke point for all inbound traffic. There's no direct path to the app server from outside — every request has to go through the WAF first, the same way it would in a real production deployment.

| Component | Role |
|---|---|
| Kali Linux | Attacker / traffic generator |
| Ubuntu Server 22.04 LTS | Target web server |
| DVWA | Intentionally vulnerable app, used as a safe attack target |
| SafeLine WAF | Nginx-based reverse proxy WAF — the control being tested |
| `/etc/hosts` | Local DNS resolution (`dvwa.local` → server IP) |
| Self-signed SSL cert | TLS termination at the WAF |

---

## Tech Stack

- **Virtualization:** VirtualBox
- **Attacker OS:** Kali Linux
- **Target OS:** Ubuntu Server 22.04 LTS
- **Web stack:** Apache2, PHP, MySQL (LAMP)
- **Target app:** [DVWA](https://github.com/digininja/DVWA)
- **Security control:** [SafeLine WAF](https://waf.chaitin.com/)
- **Encryption:** OpenSSL (self-signed cert)

---

## Installation / Lab Setup

### 1. Provision the VMs
Two VirtualBox VMs on a shared **bridged network** (so both machines act as real hosts on the LAN, not NAT):
- `KaliLinux` — 2GB RAM, 20GB disk
- `UbuntuServer` — 2GB RAM, 20GB disk, Ubuntu 22.04 LTS

Confirm connectivity between them:
```bash
ping <ubuntu_ip>   # from Kali
ping <kali_ip>     # from Ubuntu
```

### 2. Update and install base tools (Ubuntu)
```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y net-tools openssl
```

### 3. Install the LAMP stack
```bash
sudo apt-get install -y apache2 php php-mysql mysql-server
sudo mysql_secure_installation
```

### 4. Deploy DVWA
```bash
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git
sudo chown -R www-data:www-data DVWA
sudo chmod -R 755 DVWA
```
Edit `DVWA/config/config.inc.php` with your DB credentials, then create the database:
```sql
sudo mysql -u root -p
CREATE DATABASE dvwa;
CREATE USER 'dvwa_user'@'localhost' IDENTIFIED BY 'p@ssw0rd';
GRANT ALL ON dvwa.* TO 'dvwa_user'@'localhost';
FLUSH PRIVILEGES;
```
Then browse to `http://<ubuntu_ip>/DVWA/setup.php` and click **Create/Reset Database**.

### 5. Move Apache off the public ports
Edit `/etc/apache2/ports.conf`:
```
Listen 8080
```
Update `/etc/apache2/sites-available/000-default.conf` to `<VirtualHost *:8080>`, then:
```bash
sudo systemctl restart apache2
```

### 6. Set up local DNS resolution
On **both** Kali and Ubuntu, edit `/etc/hosts`:
```
<ubuntu_ip>   dvwa.local
```

### 7. Generate a self-signed SSL certificate
```bash
sudo mkdir /etc/ssl/dvwa
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/dvwa/dvwa.key \
  -out /etc/ssl/dvwa/dvwa.crt
```

### 8. Install SafeLine WAF
```bash
bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/manager.sh)" -- --en
```
Log into the SafeLine UI (default port `9443`) and:
1. Import the self-signed cert (`dvwa.crt` / `dvwa.key`).
2. Add DVWA as a protected application:
   - Domain: `www.dvwa.local`
   - Backend/reverse-proxy target: `http://<ubuntu_ip>:8080`
   - Public listener: **443 only** (remove port 80)

---

## Usage — Running the Attack Simulation

1. On Kali, browse to `https://dvwa.local` and log in to DVWA (default: `admin` / `password`).
2. Set the DVWA **Security Level** to `Low` (DVWA Security tab).
3. Go to the **SQL Injection** module and submit a classic payload:
   ```
   admin' OR '1'='1
   ```
4. Instead of a normal application response, the request should be intercepted by SafeLine WAF.
5. Check **SafeLine WAF → Logs** to confirm the request was flagged and blocked as a SQL Injection attempt, with source IP and matched rule.

---

## Results

- The SQLi payload was **blocked at the WAF layer** — it never reached Apache, PHP, or MySQL.
- The attacker received a WAF block page instead of a valid application response.
- SafeLine's logs showed the blocked request with the matched detection signature and source IP.

```
Attacker (Kali) → SafeLine WAF (443, TLS termination + inspection)
   → BLOCKED — malicious pattern matched, request dropped

Legitimate request:
User (Kali) → SafeLine WAF (443) → Apache (8080) → PHP/DVWA
   → MySQL (3306) → response returns through the same path
```

*(Add screenshots here: the SafeLine block page, and the corresponding log entry — these are the strongest proof-of-work images for this project.)*

---

## Additional WAF Controls Tested

| Control | What it does | Outcome |
|---|---|---|
| **HTTP Flood Defense** | Rate-limits requests per source to mitigate DoS/brute-force | Configured request/sec thresholds + ban duration, validated with repeated requests |
| **Authentication Gateway** | Requires credentials at the WAF before any traffic reaches the app | Confirmed WAF prompted for auth before passing traffic through |
| **Custom Deny Rules** | Manually blocks a known-bad source IP | Blocked the Kali VM's IP directly; confirmed all further requests were denied |

---

## What I Learned

- A WAF can stop a real attack without touching the vulnerable app at all — DVWA never changed, the WAF just sat in front of it. That's the same situation a lot of SOC/security teams deal with when they can't patch or rewrite an app themselves.
- Being able to trace a request through WAF → Apache → PHP → MySQL made it much easier to reason about *where* an attack was stopped, not just *that* it was stopped.
- Moving Apache off ports 80/443 wasn't just a config step — it's what forces every request through the WAF with no bypass path. That's a real architectural decision, not a checkbox.
- Don't trust the block page alone — I made a habit of confirming every block directly in SafeLine's logs.

---

## Possible Extensions

- Add [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) as a second target for broader attack coverage.
- Test XSS, command injection, and file inclusion against the same WAF policy.
- Forward SafeLine logs into a home SIEM (Wazuh / Splunk Free / ELK) for centralized alerting.
- Add an NGFW/IPS in front of the WAF for a fuller layered perimeter.

---

## License

MIT License © 2026 [Your Name]

---

## Contact

**[Your Name]** — Aspiring SOC Analyst
[LinkedIn] · [Email] · [GitHub]
