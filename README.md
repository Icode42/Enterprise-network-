# Enterprise Network Home Lab – Wazuh + Active Directory

## Overview

This project documents the design, deployment, troubleshooting, and ongoing development of a cybersecurity home lab built to simulate core enterprise infrastructure and security operations workflows.

The environment was designed to provide hands-on experience with:

* Active Directory Domain Services (AD DS)
* Windows enterprise administration
* Wazuh SIEM deployment and monitoring
* Sysmon endpoint telemetry
* Group Policy Objects (GPOs)
* Windows event collection and analysis
* Detection engineering concepts
* Linux administration and troubleshooting
* Network and DNS troubleshooting
* Threat detection and investigation

Rather than only documenting successful deployments, this repository focuses heavily on troubleshooting methodology, root-cause analysis, and operational lessons learned throughout the build process.

---

## Background

This lab was built by a veteran and cybersecurity professional currently pursuing a bachelor's degree in cybersecurity/networking. The author holds a CompTIA Security+ certification, has a background in DoD telecommunications and CAC/PKI infrastructure, and has worked as an ISSO supporting cleared environments.

This project is a combination of hands-on personal knowledge, independent research, and AI-assisted documentation (Claude by Anthropic). The lab design, troubleshooting, and security concepts were driven by the author's own experience and learning — AI and research were used as tools to supplement and document that process, the same way a practitioner might use documentation, forums, or Copilot in a professional environment.

---

## Screenshots

**Active Directory Users and Computers — lab.local OU structure with DC01**
![Active Directory](screenshots/lab_screenshot1.png)

**Group Policy Management — GPOs deployed in lab.local**
![Group Policy](screenshots/lab_screenshot2.png)

**Wazuh Threat Hunting — live alerts from DESKTOP-7SD066A**
![Wazuh Threat Hunting](screenshots/lab_screenshot3.png)

**Wazuh Endpoints Dashboard — agents registered (Windows 10 & Windows Server 2022)**
> Note: Due to host resource constraints, only one Windows VM can run alongside Wazuh at a time. This is a known lab limitation.

![Wazuh Dashboard](screenshots/lab_screenshot4.png)

---

# Technologies Used

| Technology                | Purpose                                   |
| ------------------------- | ----------------------------------------- |
| VMware Workstation Pro 17 | Virtualization platform                   |
| Windows Server            | Active Directory Domain Controller        |
| Windows 10/11             | Domain-joined endpoints                   |
| Ubuntu Server             | Wazuh Manager / SIEM infrastructure       |
| Wazuh 4.x                 | SIEM and security monitoring              |
| Sysmon                    | Advanced Windows telemetry                |
| pfSense                   | Virtual firewall and network segmentation |
| Amazon Linux              | Additional Wazuh agent testing            |
| Nmap                      | Network scanning and detection testing    |

---

# Lab Architecture

```text
                 ┌──────────────────────┐
                 │   pfSense Firewall   │
                 └──────────┬───────────┘
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
┌──────────────────┐                ┌────────────────────┐
│ Windows Endpoints │                │   Wazuh Server     │
│ Win10 / Win11     │◄──────────────►│ Manager + Indexer  │
│ Sysmon + Agent    │                │ Dashboard           │
└──────────────────┘                └────────────────────┘
        │
        │
┌──────────────────┐
│ Active Directory │
│ Domain Controller│
│ DNS + GPOs       │
└──────────────────┘
```

---

# Project Objectives

The primary goals of this lab were to:

* Build a functioning enterprise-style Active Directory environment
* Deploy and manage a SIEM platform
* Collect and analyze Windows endpoint telemetry
* Practice troubleshooting enterprise infrastructure issues
* Investigate Windows security events and process activity
* Learn detection engineering fundamentals
* Simulate reconnaissance and discovery activity
* Gain operational familiarity with Linux and Windows administration

---

# 1. Active Directory Domain Controller Deployment

## Goal

Deploy a Windows Server VM and promote it into a fully functional Domain Controller for the `lab.local` environment.

## Tasks Completed

* Installed Active Directory Domain Services (AD DS)
* Promoted Windows Server to Domain Controller
* Installed DNS during promotion
* Created forest: `lab.local`
* Configured domain authentication services
* Validated DNS functionality

---

## Issue 1 — DNS Misconfiguration After Domain Promotion

### Symptoms

* Clients could not join the domain
* `nslookup` failed
* DNS-related Event Viewer errors
* Group Policy failures

### Root Cause

The Domain Controller NIC was configured to use an external DNS server instead of itself.

### Fix

Configured the Domain Controller to use its own IP address as the primary DNS server.

### Lesson Learned

Active Directory is heavily dependent on DNS. Incorrect DNS configuration can break:

* Domain joins
* Authentication
* GPO processing
* Service discovery
* Replication

---

# 2. Organizational Unit (OU) Design

## Final OU Structure

```text
lab.local
│
├── _Admins
├── _Servers
├── _Workstations
├── _ServiceAccounts
└── _Quarantine
```

---

## Issue 2 — Group Policies Not Applying

### Symptoms

* Domain GPOs not applying
* `gpresult /r` showed only Local GPOs
* Security policies missing from endpoints

### Root Cause

Users and computers remained inside the default:

* `CN=Users`
* `CN=Computers`

containers instead of organizational units.

### Fix

Moved domain objects into the correct OUs.

### Lesson Learned

Group Policies apply to organizational units, not default containers.

---

# 3. Group Policy Management

## GPOs Created

* Workstation Security Baseline
* Server Security Baseline
* Domain Controller Baseline
* Login Banner Policy
* Disable USB Storage
* Wallpaper Policy
* Planned Sysmon deployment via GPO

---

## Issue 3 — Workstation GPO Failures

### Symptoms

* Wallpaper policy not applying
* Security baselines missing
* `gpresult /r` showed no domain policies

### Troubleshooting Steps

* Verified domain join
* Ran `gpupdate /force`
* Checked SYSVOL accessibility
* Reviewed Event Viewer logs
* Validated DNS settings

### Root Cause

The workstation was using router-based DNS instead of the Domain Controller.

### Fix

Configured the workstation to use the Domain Controller as its primary DNS server.

### Lesson Learned

Most Group Policy failures trace back to:

* DNS misconfiguration
* Incorrect OU placement
* SYSVOL issues

---

## Issue 4 — SYSVOL Replication / Missing Policies

### Symptoms

* Missing policy templates
* GPO editor errors
* SYSVOL partially populated

### Root Cause

SYSVOL did not fully initialize after Domain Controller promotion.

### Fix

Restarted replication services:

```powershell
net stop ntfrs
net start ntfrs
```

Validated replication status:

```powershell
ntfrsutl ds
```

### Lesson Learned

SYSVOL health is critical for:

* Group Policy processing
* Login scripts
* Baseline deployment
* Enterprise consistency

---

## Issue 5 — Login Banner Policy Not Displaying

### Symptoms

* Legal notice did not appear before login

### Root Cause

GPO inheritance prevented the policy from applying.

### Fix

Enabled **Enforced** on the Login Banner GPO.

### Lesson Learned

Understanding inheritance and precedence is essential when troubleshooting Group Policy.

---

# 4. Domain Join Troubleshooting

## Issue 6 — Windows Client Could Not Join Domain

### Symptoms

* "Cannot contact domain controller"
* "The specified domain does not exist"

### Root Cause

The client used DHCP/router DNS and lacked the proper DNS suffix.

### Fix

Configured:

* Primary DNS = Domain Controller IP
* DNS Suffix = `lab.local`

Rejoined the system to the domain.

### Lesson Learned

DNS is the foundation of Active Directory communication.

---

## Issue 7 — Domain Accounts Unable to Log In

### Symptoms

* "Incorrect username or password"
* Domain credentials rejected

### Root Cause

Broken workstation trust relationship with the domain.

### Fix

* Reset computer account in Active Directory
* Removed workstation from domain
* Rejoined the domain

### Lesson Learned

Resetting the computer account resolves many trust relationship issues.

---

# 5. Wazuh SIEM Deployment

## Components

* Wazuh Manager
* Wazuh Indexer
* Wazuh Dashboard
* Ubuntu Server
* Windows endpoints
* Sysmon telemetry

---

## Issue 8 — Wazuh Agent DNS Resolution Failure

### Symptoms

```text
Remote name could not be resolved
```

### Root Cause

The Windows VM used internal DNS that could not resolve the Wazuh Manager.

### Fix

Configured Google DNS (`8.8.8.8`) as a fallback resolver.

### Lesson Learned

Agent enrollment and SIEM communication depend heavily on functional DNS.

---

## Issue 9 — Wazuh Dashboard and Indexer Instability

### Symptoms

* Dashboard timeouts
* Failed host connections
* Intermittent GUI availability

### Troubleshooting Commands

```bash
systemctl status wazuh-manager
systemctl status wazuh-indexer
systemctl status wazuh-dashboard
free -h
df -h
```

### Root Cause

Insufficient VM resources caused service instability.

### Fix

Restarted Wazuh services:

```bash
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-dashboard
```

### Lesson Learned

SIEM platforms are resource-intensive and require careful monitoring of:

* RAM usage
* Disk utilization
* Service health
* Index growth

---

# 6. Sysmon Integration and Endpoint Telemetry

## Issue 10 — Sysmon Logs Missing From Wazuh

### Root Cause

The Sysmon event channel was missing from the Wazuh agent configuration.

### Fix

Added the following configuration to `ossec.conf`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Restarted the Wazuh agent:

```powershell
NET STOP WazuhSvc
NET START WazuhSvc
```

---

## Sysmon Event Investigation

### Event Types Reviewed

| Event ID | Description           |
| -------- | --------------------- |
| 1        | Process Creation      |
| 3        | Network Connection    |
| 11       | File Creation         |
| 13       | Registry Modification |

### Fields Investigated

* `process.name`
* `commandLine`
* `win.system.eventID`
* `user.name`
* `parentProcess`
* Hash values

### Example Activities Observed

* `net.exe` account discovery
* User/group modifications
* Process execution telemetry
* Network connection events
* Discovery-related activity

### MITRE ATT&CK Examples

| Activity   | Technique                         |
| ---------- | --------------------------------- |
| `net user` | T1087 – Account Discovery         |
| Sudo usage | T1548.003 – Sudo and Sudo Caching |

### Lesson Learned

Sysmon significantly improves Windows visibility and provides valuable telemetry for:

* Threat hunting
* Detection engineering
* Process analysis
* Security investigations

---

# 7. Network Scanning and Detection Testing

## Tools Used

* Nmap
* Sysmon
* Wazuh
* auditd (Linux)

## Activities Tested

```bash
nmap -sT <target>
nmap -sS <target>
nmap -sS -T5 <target>
```

## Findings

* Full TCP scans generated more visible telemetry
* SYN stealth scans produced limited host-level visibility
* Event ID 3 network logging required proper Sysmon configuration
* Linux monitoring required auditd and additional logging configuration

### Linux Audit Rule Example

```bash
sudo auditctl -w /usr/bin/nmap -p x -k nmap_scan
```

### Lesson Learned

Host telemetry alone may not fully detect stealth scans. Network-based monitoring tools such as:

* Suricata
* Zeek
* Snort

provide deeper packet-level visibility.

---

# Skills Practiced

## Windows Administration

* Active Directory administration
* OU design
* DNS troubleshooting
* Group Policy management
* Domain joins
* Trust relationship repair

## Linux Administration

* Service troubleshooting
* Package management
* Resource monitoring
* auditd configuration
* Wazuh administration

## Security Operations

* SIEM monitoring
* Log analysis
* Event investigation
* Threat hunting
* Detection validation
* MITRE ATT&CK mapping
* Endpoint telemetry analysis

---

# Future Improvements

## Short-Term Goals

* Tune Sysmon rules
* Create custom Wazuh detection rules
* Expand Linux monitoring
* Simulate attacks with Atomic Red Team
* Improve dashboard visualizations

## Long-Term Goals

* Deploy Security Onion
* Add Suricata IDS
* Integrate Elastic or Splunk
* Automate deployment with Ansible/Terraform
* Build a full SOC-style monitoring workflow

---

# Key Lessons Learned

* DNS is the backbone of Active Directory and SIEM communication
* Group Policy issues often originate from DNS or OU placement
* SYSVOL health is essential for domain stability
* Sysmon dramatically improves endpoint visibility
* SIEM platforms require careful resource planning
* Hands-on troubleshooting builds real operational experience
* Detection engineering depends on understanding both telemetry and logging limitations

---

# Final Thoughts

This project was designed to move beyond basic tool installation and focus on operational troubleshooting, visibility, and real-world security workflows.

The most valuable part of the lab was learning how enterprise systems fail, how telemetry flows through the environment, and how to methodically troubleshoot issues across Windows, Linux, networking, and SIEM infrastructure.

The environment will continue evolving with additional detection engineering, network monitoring, automation, and threat simulation capabilities.
