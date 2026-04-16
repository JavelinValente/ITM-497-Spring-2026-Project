# Wazuh as a SIEM Tool

> **Framework Section:** Platform Selection & Implementation
> **Audience:** SMB IT Administrators, vCISOs, Security Engineers
> **Skill Level:** Beginner to Intermediate

---

## Overview

Wazuh is a free, open-source security platform that provides unified XDR and SIEM capabilities. It is one of the most accessible SIEM solutions for SMBs due to its zero licensing cost, active community, and broad integration support. Wazuh can be deployed on-premises, in the cloud, or in hybrid environments.

| Attribute | Details |
|---|---|
| **Type** | Open-source XDR / SIEM |
| **License** | Free (GPLv2 / Apache 2.0) |
| **Deployment** | On-premises, Cloud, Hybrid |
| **Agent Support** | Windows, Linux, macOS, Solaris, AIX, HP-UX |
| **Official Docs** | https://documentation.wazuh.com |
| **GitHub** | https://github.com/wazuh/wazuh |
| **Community** | https://community.wazuh.com |

---

## 1. Installation & Setup

Wazuh consists of three core components:

- **Wazuh Indexer** — based on OpenSearch; stores and indexes security events
- **Wazuh Server** — processes logs, runs detection rules, manages agents
- **Wazuh Dashboard** — web UI for visibility, alerts, and compliance reporting

### Deployment Options

| Option | Best For | Notes |
|---|---|---|
| **All-in-One** | SMBs with < 500 endpoints | Single server, easiest to manage |
| **Distributed** | Larger or HA environments | Separate indexer/server/dashboard nodes |
| **Wazuh Cloud** | Teams without infrastructure | Managed SaaS, no self-hosting required |

### Minimum Hardware Requirements (All-in-One)

| Component | CPU | RAM | Storage |
|---|---|---|---|
| Wazuh Indexer | 4 cores | 8 GB | 200 GB+ SSD |
| Wazuh Server | 2 cores | 4 GB | 50 GB+ |
| Wazuh Dashboard | 2 cores | 4 GB | Shared with server |

### Key Resources

- [Quickstart Guide (All-in-One)](https://documentation.wazuh.com/current/quickstart.html)
- [Installation Guide (Distributed)](https://documentation.wazuh.com/current/installation-guide/index.html)
- [Wazuh Cloud Documentation](https://documentation.wazuh.com/current/cloud-service/index.html)
- [System Requirements](https://documentation.wazuh.com/current/installation-guide/requirements.html)
- [Deployment on Docker](https://documentation.wazuh.com/current/deployment-options/docker/index.html)
- [Deployment on Kubernetes](https://documentation.wazuh.com/current/deployment-options/deploying-with-kubernetes/index.html)

---

## 2. Log Collection & Agents

Wazuh agents are lightweight programs deployed on endpoints that collect logs, monitor file integrity, detect vulnerabilities, and report to the Wazuh Server. Agentless collection via syslog is also supported for network devices.

### Agent Deployment Methods

- Package manager (apt, yum, dnf)
- Group Policy Object (GPO) for Windows environments
- Configuration management tools (Ansible, Chef, Puppet)
- Auto-enrollment for large-scale deployments

> Agents communicate over encrypted channels on **port 1514 TCP/UDP** by default.

### Supported Log Sources

| Category | Sources |
|---|---|
| **Windows** | Event Logs (Security, System, Application), Sysmon, PowerShell logs |
| **Linux** | syslog, auditd, journald, auth.log |
| **macOS** | Unified Log |
| **Network Devices** | Syslog from firewalls, routers, switches (agentless) |
| **Cloud - AWS** | CloudTrail, VPC Flow Logs, GuardDuty, S3 access logs |
| **Cloud - Azure** | Activity Logs, Entra ID (Azure AD), Microsoft Defender |
| **Cloud - GCP** | Audit Logs, Cloud Logging |
| **Microsoft 365** | Audit logs, sign-in logs via Microsoft Graph API |

### Key Resources

- [Agent Enrollment & Deployment](https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/index.html)
- [Log Data Collection Overview](https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/index.html)
- [AWS Security Monitoring](https://documentation.wazuh.com/current/cloud-security/amazon/index.html)
- [Azure Security Monitoring](https://documentation.wazuh.com/current/cloud-security/azure/index.html)
- [Microsoft 365 Integration](https://documentation.wazuh.com/current/cloud-security/office365/index.html)
- [Agentless Monitoring](https://documentation.wazuh.com/current/user-manual/capabilities/agentless-monitoring/index.html)

---

## 3. Detection Rules & Alerts

Wazuh ships with **3,000+ out-of-the-box detection rules** mapped to the MITRE ATT&CK framework. Rules are written in XML and can be customized or extended.

### Rule Basics

| Item | Details |
|---|---|
| **Default rule location** | `/var/ossec/ruleset/rules/` |
| **Custom rule location** | `/var/ossec/etc/rules/local_rules.xml` |
| **Severity scale** | 0 (informational) → 15 (critical) |
| **Alert triggers** | Email, Slack/Teams webhook, syslog, custom integrations |

### Alert Level Reference

| Level | Severity | Example Use |
|---|---|---|
| 0–3 | Informational | Successful logins, routine events |
| 4–7 | Low / Medium | Failed logins, policy violations |
| 8–11 | High | Brute force, privilege escalation attempts |
| 12–15 | Critical | Active exploitation, rootkit detection |

### MITRE ATT&CK Coverage

Wazuh maps rules natively to MITRE ATT&CK tactics and techniques, visible directly in the dashboard. Key tactic coverage includes:

- Initial Access
- Execution & Persistence
- Credential Access
- Lateral Movement
- Defense Evasion
- Exfiltration

### Key Resources

- [Ruleset Overview](https://documentation.wazuh.com/current/user-manual/ruleset/index.html)
- [Custom Rules Guide](https://documentation.wazuh.com/current/user-manual/ruleset/custom-rules.html)
- [MITRE ATT&CK Integration](https://documentation.wazuh.com/current/user-manual/capabilities/mitre.html)
- [Alert Configuration](https://documentation.wazuh.com/current/user-manual/manager/alert-threshold.html)
- [Email Notification Setup](https://documentation.wazuh.com/current/user-manual/manager/manual-email-report/index.html)
- [Slack / Webhook Integration](https://documentation.wazuh.com/current/user-manual/manager/manual-integration.html)

---

## 4. Compliance Modules

Wazuh includes built-in compliance modules for major regulatory frameworks, helping SMBs demonstrate adherence without expensive third-party tooling.

### Supported Frameworks

| Framework | Coverage Areas | Dashboard |
|---|---|---|
| **PCI DSS** | File integrity, log review, access control, vulnerability management | ✅ |
| **HIPAA** | Audit controls, access management, data integrity, breach detection | ✅ |
| **NIST 800-53** | Configuration management, access control, incident response | ✅ |
| **SOC 2** | Availability, security, confidentiality, processing integrity | ✅ |
| **GDPR** | Data access controls, breach detection, data subject rights | ✅ |
| **CIS Benchmarks** | OS and application hardening checks (Linux, Windows, macOS) | ✅ |
| **TSC** | Trust Services Criteria for SOC audits | ✅ |

### Key Resources

- [PCI DSS Compliance Module](https://documentation.wazuh.com/current/compliance/pci-dss/index.html)
- [HIPAA Compliance Module](https://documentation.wazuh.com/current/compliance/hipaa/index.html)
- [NIST 800-53 Module](https://documentation.wazuh.com/current/compliance/nist/index.html)
- [GDPR Module](https://documentation.wazuh.com/current/compliance/gdpr/index.html)
- [CIS-CAT Integration](https://documentation.wazuh.com/current/user-manual/capabilities/policy-monitoring/ciscat/ciscat-configuration.html)
- [SCA (Security Configuration Assessment)](https://documentation.wazuh.com/current/user-manual/capabilities/policy-monitoring/sca/index.html)

---

## Additional Resources

| Resource | Link |
|---|---|
| Wazuh Blog | https://wazuh.com/blog |
| Wazuh YouTube Channel | https://www.youtube.com/@wazuh |
| GitHub Issues / Releases | https://github.com/wazuh/wazuh/releases |
| Community Forum | https://community.wazuh.com |
| Slack Community | https://wazuh.com/community/join-us-on-slack |

---

*Back to [SIEM Framework Index](../README.md)*
