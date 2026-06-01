# Website Security Incident Case Study

This repository contains a real-world Website security incident investigation and mitigation case study involving large-scale XML-RPC attacks, Apache resource exhaustion, and Out-Of-Memory (OOM) events.

The objective of this case study is to demonstrate the investigation process, root cause analysis, and security controls implemented to stabilize the production environment.

---

## 📌 **Purpose of the Repository**

### 1. Website Incident Analysis

**File:**

```text
Website-incident-analysis.md
```

**Overview:**

This article focuses on the investigation and troubleshooting process used to identify the root cause of service degradation and outages.

Topics covered:

- Linux system investigation
- Memory utilization analysis
- OOM Killer detection
- Apache log analysis
- XML-RPC attack identification
- SSH attack investigation
- Network and service verification
- Real-time monitoring techniques

**Key Skills Demonstrated:**

- Incident Response
- Linux Administration
- Log Analysis
- Troubleshooting Methodology
- Root Cause Analysis

---

### 2. Website XML-RPC Attack Defense

**File:**

```text
Website-xmlrpc-attack-defense.md
```

**Overview:**

This article explains how a large-scale XML-RPC attack was mitigated using Apache security controls and automated threat prevention mechanisms.

Topics covered:

- XML-RPC attack analysis
- Distributed botnet behavior
- Challenges with IP-based blocking
- Apache Rate Limiting
- mod_evasive configuration
- Fail2Ban implementation
- Security hardening recommendations
- Lessons learned

**Key Skills Demonstrated:**

- Web Security
- Apache Administration
- Threat Mitigation
- Fail2Ban Configuration
- Security Hardening
- Production Incident Management

---

## Incident Summary

### Attack Statistics

| Metric | Value |
|----------|----------|
| Attack Type | Website XML-RPC Brute Force / DDoS |
| Total Requests | 1,108,462+ |
| Infrastructure | Apache + MariaDB + Website |
| Impact | OOM Events, Service Disruption |
| Threat Source | Distributed Botnet |
| Severity | High |

---

## Investigation Workflow

```text
Performance Issue
        │
        ▼
Memory Analysis
        │
        ▼
OOM Investigation
        │
        ▼
Apache Log Review
        │
        ▼
XML-RPC Detection
        │
        ▼
Root Cause Confirmation
        │
        ▼
Mitigation Implementation
```

---

## Security Controls Implemented

### Detection

- Apache Access Log Analysis
- System Journal Analysis
- OOM Killer Investigation
- Authentication Log Monitoring

### Prevention

- Apache Rate Limiting
- mod_evasive
- Fail2Ban
- XML-RPC Protection

### Recommended Enhancements

- Cloudflare WAF
- AWS WAF
- Managed Bot Protection
- Multi-Factor Authentication
- Continuous Security Monitoring

---

## Technologies Used

- Linux
- Apache2
- MariaDB
- Wordpress
- Fail2Ban
- mod_evasive
- UFW
- AWS EC2

---

Areas of Expertise:

- AWS Cloud Infrastructure
- Linux Administration
- CI/CD Automation
- Kubernetes
- Docker
- Security Hardening
- Performance Optimization
- Production Support

---

## Disclaimer

The IP addresses, attack samples, and log entries presented in these articles are included solely for educational and security awareness purposes.

All recommendations should be tested in non-production environments before deployment to production systems.
