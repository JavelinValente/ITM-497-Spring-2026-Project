# 🛡️ SMB Security Visibility & SIEM Development Framework

> A practical, opinionated framework for small and medium-sized businesses (SMBs) looking to build a Security Information and Event Management (SIEM) program and improve overall security visibility — without enterprise-level budgets.

---

## 📌 Who This Is For

This repository is designed for:
- IT administrators and security engineers at SMBs (typically 50–1,000 employees)
- vCISOs and consultants helping SMBs mature their security programs
- MSPs/MSSPs building repeatable security frameworks for clients
- Anyone starting from limited or no centralized logging/visibility

---

## 🗂️ Table of Contents

1. [Section 1 – Assessment & Readiness](#section-1--assessment--readiness)
2. [Section 2 – Define Objectives & Use Cases](#section-2--define-objectives--use-cases)
3. [Section 3 – Log Source Strategy](#section-3--log-source-strategy)
4. [Section 4 – SIEM Platform Selection](#section-4--siem-platform-selection)
5. [Section 5 – Detection Engineering](#section-5--detection-engineering)
6. [Section 6 – Incident Response Integration](#section-6--incident-response-integration)
7. [Section 7 – Roles & Responsibilities](#section-7--roles--responsibilities)
8. [Section 8 – Operationalization & Tuning](#section-8--operationalization--tuning)
9. [Section 9 – Metrics & Reporting](#section-9--metrics--reporting)
10. [Section 10 – Roadmap & Maturity Growth](#section-10--roadmap--maturity-growth)

---

## Section 1 – Assessment & Readiness

Before deploying any tooling, you need a clear picture of your current environment. Rushing into a SIEM without this step leads to noisy, untuned alerts and wasted budget.

### 1.1 Current Security Posture Review
- Conduct an internal gap analysis against a baseline framework (CIS Controls v8, NIST CSF, or ISO 27001)
- Identify what security controls are already in place (EDR, firewall, MFA, etc.)
- Document known weaknesses and previously identified incidents
- Review any existing logging — what's already being captured and where is it going?

### 1.2 Asset Inventory
You can't monitor what you don't know you have. Build a complete inventory:

| Asset Type | Examples | Priority |
|---|---|---|
| Endpoints | Workstations, laptops | High |
| Servers | On-prem, cloud VMs | Critical |
| Network Devices | Firewalls, switches, VPN | Critical |
| Cloud Services | AWS, Azure, M365, GCP | High |
| SaaS Applications | Salesforce, Okta, GitHub | Medium–High |
| OT/IoT | Printers, cameras, HVAC controllers | Medium |

### 1.3 Identify Existing Log Sources
- Map which systems are generating logs and whether those logs are currently retained
- Determine log format (syslog, JSON, CEF, LEEF, Windows Event Log, etc.)
- Identify gaps — critical systems with no logging configured

### 1.4 Compliance Requirements
Knowing your compliance obligations helps prioritize what to log and retain:
- **PCI-DSS** – Requires 12 months log retention, access monitoring, change detection
- **HIPAA** – Requires audit controls and activity review for ePHI access
- **SOC 2** – Requires logging of access to systems with customer data
- **CMMC / DFARS** – Required for DoD contractors; specifies specific log events
- **State Privacy Laws (CCPA, NYDFS)** – May require breach detection and response capabilities

### 1.5 Budget & Resource Constraints
SMB SIEM programs must be realistic. Assess:
- Available budget (licensing, infrastructure, staff time)
- Internal staffing capacity — do you have anyone to respond to alerts?
- Whether a co-managed or fully managed model (MDR/MSSP) makes more sense
- Tolerance for cloud-hosted vs. on-premise solutions

> **💡 Tip:** If you have fewer than 2 dedicated security staff, seriously evaluate MDR/MSSP options before building in-house.

---

## Section 2 – Define Objectives & Use Cases

A SIEM without defined use cases is just an expensive log storage system. Define what you're trying to detect *before* you deploy.

### 2.1 Threat Scenarios to Prioritize

For most SMBs, these are the highest-impact threat scenarios to start with:

| Threat Scenario | Why It Matters for SMBs |
|---|---|
| Ransomware deployment | Most common cause of catastrophic SMB incidents |
| Business Email Compromise (BEC) | High financial impact, often undetected for weeks |
| Credential stuffing / brute force | Common initial access vector |
| Insider threat / data exfiltration | Departing employees remain a top risk |
| Lateral movement | Attackers pivoting after initial foothold |
| Cloud account compromise | Critical as SMBs adopt SaaS and IaaS |
| Phishing / malicious email delivery | Primary initial access vector |

### 2.2 Defining Visibility Goals

Answer these questions as a team:
- What would we **need to know** to detect a breach within 24 hours?
- What events would indicate a credential was compromised?
- Do we have visibility into privileged account activity?
- Can we detect data leaving our environment?

### 2.3 Measurable Goals

Set baseline targets before go-live, even if estimated:

| Metric | Starting Target |
|---|---|
| Mean Time to Detect (MTTD) | < 24 hours for critical alerts |
| Mean Time to Respond (MTTR) | < 4 hours after alert triage |
| False Positive Rate | < 20% of total alerts |
| Log Source Coverage | 80%+ of critical systems ingesting within 90 days |

---

## Section 3 – Log Source Strategy

Garbage in, garbage out. Thoughtful log source selection is the foundation of any effective SIEM.

### 3.1 Critical Log Sources (Tier 1 — Do These First)

| Log Source | What to Capture | Why |
|---|---|---|
| Active Directory / Entra ID | Logins, failed auth, group changes, privilege escalation | Identity is the perimeter |
| Firewall / NGFW | Allow/deny traffic, outbound connections, VPN auth | Network visibility |
| EDR (CrowdStrike, Defender, SentinelOne) | Process execution, file changes, network connections | Endpoint telemetry |
| Cloud Identity (Okta, Azure AD, Google Workspace) | SSO events, MFA failures, account changes | SaaS access control |
| Email Gateway | Phishing detections, attachment analysis, delivery logs | Primary attack vector |
| VPN / Remote Access | Auth events, session duration, unusual geos | Remote access monitoring |

### 3.2 Secondary Log Sources (Tier 2 — Add After Baseline)

- DNS logs (internal resolver and external queries)
- DHCP logs (IP-to-hostname mapping for correlation)
- Web proxy / Secure Web Gateway
- Cloud infrastructure logs (AWS CloudTrail, Azure Activity Log, GCP Audit)
- SaaS application logs (M365 Unified Audit Log, Salesforce Event Log)
- PAM/privileged access tools

### 3.3 Log Forwarding Methods

| Method | Use Case |
|---|---|
| Syslog (UDP/TCP 514) | Network devices, Linux systems |
| Windows Event Forwarding (WEF) | Windows endpoints and servers |
| API / Webhook integration | SaaS platforms (M365, Okta, etc.) |
| Agent-based (Beats, NXLog, Splunk UF) | Servers and endpoints requiring custom parsing |
| Cloud-native connectors | AWS, Azure, GCP log forwarding to SIEM |

### 3.4 Log Retention Policy

| Data Category | Recommended Retention | Notes |
|---|---|---|
| Authentication logs | 12 months | Often required by compliance frameworks |
| Network flow data | 90 days | Storage-intensive; compress older data |
| Endpoint telemetry | 90–180 days | Balance between storage and investigation depth |
| Cloud activity logs | 12 months | Critical for cloud incident investigations |
| Raw log archives | 12–24 months cold | Compressed, offline or object storage |

### 3.5 Log Normalization & Enrichment

Raw logs are often inconsistent across sources. Standardize them by:
- Normalizing timestamps to UTC
- Mapping source IPs to asset names and owners (asset enrichment)
- Tagging logs with environment (prod/dev/cloud) and criticality
- Integrating threat intel feeds for IP/domain reputation tagging
- Adding geolocation data to authentication events

---

## Section 4 – SIEM Platform Selection

### 4.1 Build vs. Buy vs. MDR

| Option | Pros | Cons | Best For |
|---|---|---|---|
| Build In-House | Full control, no licensing markup | Requires dedicated staff and expertise | Orgs with 2+ security FTEs |
| SaaS SIEM | Faster deployment, vendor-managed infra | Recurring cost, less customization | Most SMBs |
| MSSP/MDR | 24/7 coverage, expertise included | Less visibility into operations, higher cost | Orgs with no security staff |
| Hybrid (SIEM + MDR) | Internal visibility + external SOC | More complex vendor management | Growing security programs |

### 4.2 SMB-Friendly SIEM Platforms

| Platform | Deployment | Cost Model | SMB Fit | Notes |
|---|---|---|---|---|
| **Microsoft Sentinel** | Cloud (Azure) | Pay-per-GB ingestion | High (if already M365) | Deep M365 integration; cost can spike |
| **Elastic SIEM (ELK)** | Self-hosted or cloud | Open source + support | High | Strong community; requires tuning expertise |
| **Wazuh** | Self-hosted | Free / open source | High | Great for lean budgets; active community |
| **Splunk Cloud** | SaaS | EPS or data ingest | Medium | Powerful but expensive at scale |
| **IBM QRadar** | On-prem or SaaS | EPS-based | Medium | Feature-rich; complex for small teams |
| **Securonix** | SaaS | User-based | Medium | Strong UEBA capabilities |
| **LogRhythm Axon** | SaaS | User-based | Medium | Simplified interface vs. classic LogRhythm |

### 4.3 Evaluation Criteria

Score each platform against these criteria before selection:

- [ ] Native integrations with your existing stack (EDR, IdP, cloud)
- [ ] Out-of-the-box detection content quality
- [ ] Ease of writing custom detection rules
- [ ] Licensing model predictability (avoid surprise bills)
- [ ] Total cost of ownership including staff time
- [ ] API access and SOAR compatibility
- [ ] Vendor roadmap and community health

---

## Section 5 – Detection Engineering

Deploying a SIEM is easy. Building detections that actually fire on real attacks — without flooding analysts with noise — is the hard part.

### 5.1 Start with OOTB Rules, Then Tune

- Enable vendor-provided out-of-the-box (OOTB) detection rules immediately
- Run in *alert* mode (not block mode) for 2–4 weeks to understand baseline noise
- Document and suppress known-good false positives with evidence
- Gradually layer in custom rules as your environment understanding grows

### 5.2 MITRE ATT&CK Alignment

Map every detection rule to a MITRE ATT&CK technique. This allows you to:
- Identify gaps in coverage across the attack lifecycle
- Prioritize rule development based on your threat profile
- Communicate detection coverage to leadership using a visual heatmap

**Recommended starting techniques to cover:**

| Tactic | Technique | Priority |
|---|---|---|
| Initial Access | Phishing (T1566), Valid Accounts (T1078) | Critical |
| Execution | PowerShell (T1059.001), WMI (T1047) | High |
| Persistence | Scheduled Tasks (T1053), Registry Run Keys (T1547) | High |
| Privilege Escalation | Token Impersonation (T1134) | High |
| Credential Access | Brute Force (T1110), LSASS Dump (T1003.001) | Critical |
| Lateral Movement | Pass-the-Hash (T1550.002), RDP (T1021.001) | Critical |
| Exfiltration | Data to Cloud Storage (T1567) | High |
| Impact | Data Encrypted for Impact / Ransomware (T1486) | Critical |

### 5.3 Alert Severity Tiers

| Tier | Response Time | Examples |
|---|---|---|
| **P1 – Critical** | Immediate (< 15 min) | Ransomware IOC, active credential dump, C2 callback |
| **P2 – High** | < 1 hour | Brute force success, impossible travel, privileged account anomaly |
| **P3 – Medium** | < 4 hours | Multiple failed logins, suspicious scheduled task, new admin account |
| **P4 – Low** | Next business day | Policy violation, unusual process, informational enrichment |

### 5.4 Tuning Discipline

- Document every suppression with justification and review date
- Set a quarterly rule review cycle — delete rules with zero hits in 90 days
- Track false positive rate per rule; any rule over 80% FP rate needs immediate rework
- Use allow-lists for known-good activity (IT scanner IPs, service accounts, etc.)

---

## Section 6 – Incident Response Integration

A SIEM that generates alerts with no response process is security theater. Detection and response must be designed together.

### 6.1 Escalation Paths

Define clear escalation paths before your first alert fires:

```
Alert Fires
    │
    ▼
Tier 1 Analyst (or on-call IT) reviews → False Positive? → Suppress + Document
    │
    ▼ Confirmed Suspicious
Tier 2 / Senior Analyst investigates
    │
    ▼ Confirmed Incident
Incident Commander declared → IR Plan activated
    │
    ▼
Executive / Legal notification (if required)
```

### 6.2 Core Playbooks to Build First

Develop documented playbooks for your top alert types before going live:

| Playbook | Trigger | Key Steps |
|---|---|---|
| Compromised Credentials | Impossible travel, credential stuffing success | Force password reset, revoke sessions, check for persistence |
| Ransomware Detection | Mass file encryption, known IOC, shadow copy deletion | Isolate host, notify IR lead, assess blast radius |
| Phishing Response | User report or email gateway detection | Quarantine email, check for clicks, hunt for lateral movement |
| Privilege Escalation | New admin account, group membership change | Validate with change management, investigate if unauthorized |
| Data Exfiltration | Large outbound transfer, cloud upload anomaly | Block/quarantine, preserve evidence, assess what was taken |
| Malware Execution | EDR alert, suspicious process | Isolate endpoint, collect forensic triage, determine scope |

### 6.3 Tooling Integration

- **Ticketing:** Integrate alerts with your ticketing system (Jira, ServiceNow, or even Linear) for tracking and SLA enforcement
- **Communication:** Define an incident communication channel (Slack, Teams) with clear naming conventions (e.g., `#incident-2024-001`)
- **Evidence Preservation:** Establish log evidence retention separate from operational retention (legal hold process)
- **Forensics:** Pre-identify your forensic toolset (Velociraptor, KAPE, FTK Imager) and train staff before an incident

---

## Section 7 – Roles & Responsibilities

### 7.1 SIEM Ownership Models

| Model | Description | Best For |
|---|---|---|
| Internal IT Owner | Single IT admin owns the SIEM alongside other duties | Very small orgs; creates burnout risk |
| Dedicated Security Analyst | One or more FTEs focused on SIEM operations | Orgs with 200+ employees or high-risk profiles |
| vCISO-Led | Fractional security leader manages strategy; IT handles ops | Orgs needing strategic guidance without full-time CISO |
| Co-Managed SIEM (MDR) | Vendor provides 24/7 monitoring; internal team handles IR | Most SMBs; recommended for < 2 security FTEs |

### 7.2 RACI Matrix

| Activity | Internal IT | Security Analyst | vCISO | MSSP/MDR |
|---|---|---|---|---|
| SIEM platform management | R | A | C | I |
| Detection rule development | C | R | A | C |
| Alert triage (Tier 1) | I | R | I | A |
| Incident investigation | C | R | A | C |
| Executive reporting | I | C | R | I |
| Compliance reporting | C | R | A | I |
| Vendor management | R | C | A | I |

*R=Responsible, A=Accountable, C=Consulted, I=Informed*

### 7.3 Training Requirements

All personnel involved in SIEM operations should complete:
- [ ] Foundational: CompTIA Security+ or equivalent
- [ ] SIEM-specific: Platform vendor training (Splunk Fundamentals, Elastic for Security, etc.)
- [ ] Detection: MITRE ATT&CK Navigator familiarization
- [ ] IR: Annual tabletop exercise participation
- [ ] Recommended: SANS SEC555 (SIEM with Tactical Analytics) for dedicated analysts

---

## Section 8 – Operationalization & Tuning

Going live is not the finish line — it's the starting gun. Ongoing operations are where programs succeed or fail.

### 8.1 Daily Operations Checklist

```
[ ] Review open P1/P2 alerts from prior 24 hours
[ ] Check log source health dashboard — any sources silent or degraded?
[ ] Review overnight automated reports for anomalies
[ ] Triage any new medium-severity alerts
[ ] Document any new suppressions or tuning changes
```

### 8.2 Weekly Operations

- Review false positive rates per rule and tune as needed
- Check for new threat intelligence relevant to your industry
- Review user access anomalies and dormant account reports
- Update asset inventory with any new systems added during the week

### 8.3 Monthly Operations

- Full alert backlog review and closure of stale tickets
- Rule performance review — remove or rework underperforming rules
- Threat intel feed review and IOC refresh
- Log source audit — confirm all critical sources are actively ingesting
- Patch and update SIEM platform and agents

### 8.4 Quarterly Operations

- Purple team or tabletop exercise to validate detection coverage
- Full MITRE ATT&CK coverage review and gap analysis
- Playbook review and updates
- Executive security briefing with metrics and roadmap update
- Compliance posture review

### 8.5 Threat Intelligence Integration

Integrating threat intel transforms your SIEM from reactive to proactive:

| Intel Type | Sources | Integration Method |
|---|---|---|
| IP/Domain Reputation | VirusTotal, AbuseIPDB, AlienVault OTX | API enrichment on network logs |
| Malware IOCs (hashes) | MISP, MalwareBazaar, vendor feeds | Lookup tables in detection rules |
| TTPs | MITRE ATT&CK, ISAC feeds | Inform detection rule development |
| Vulnerability Intel | NVD, vendor advisories | Correlate with asset inventory |
| Industry-specific | FS-ISAC, H-ISAC, MS-ISAC (free for gov/edu) | Human review + SIEM integration |

---

## Section 9 – Metrics & Reporting

If you can't measure it, you can't improve it — and you can't fund it.

### 9.1 Operational KPIs

Track these metrics weekly/monthly for continuous improvement:

| KPI | Definition | Target |
|---|---|---|
| **Alert Volume** | Total alerts generated per day/week | Establish baseline; watch for spikes |
| **True Positive Rate** | % of alerts confirmed as real threats | > 25% (anything lower = too noisy) |
| **False Positive Rate** | % of alerts determined to be benign | < 30% for tuned environment |
| **MTTD** | Mean Time to Detect (from event to alert) | < 24 hours for critical scenarios |
| **MTTR** | Mean Time to Respond (from alert to containment) | < 4 hours for P1 incidents |
| **Log Source Coverage** | % of critical assets actively sending logs | > 90% |
| **Dwell Time** | Average time attacker was in environment before detection | < 48 hours target |

### 9.2 Executive Reporting

Leadership doesn't need raw alert counts — they need business context. Monthly executive reports should include:

- **Risk posture summary** — Are we better or worse than last month? Why?
- **Incident summary** — Count and severity of confirmed incidents, outcomes
- **Top threats observed** — What attack types are targeting the org?
- **Coverage improvements** — New log sources, new rules deployed
- **Open risks** — Gaps in coverage with business impact context and remediation timeline
- **Budget vs. roadmap** — Are we on track with the planned program?

### 9.3 Compliance Reporting

Most frameworks require evidence of logging and monitoring. Maintain automated or semi-automated reports for:

- Privileged account activity and access reviews (SOX, PCI, SOC 2)
- Failed authentication reports (all frameworks)
- System change logs (PCI DSS Req. 10)
- Data access logs for sensitive systems (HIPAA, CMMC)
- Evidence of log review activity (documented in ticketing system)

---

## Section 10 – Roadmap & Maturity Growth

### 10.1 SIEM Maturity Levels

| Level | Description | Characteristics |
|---|---|---|
| **1 – Initial** | Ad-hoc logging, reactive | No centralized logging; detection is accidental |
| **2 – Developing** | Centralized logging, basic alerting | Core log sources ingesting; OOTB rules enabled |
| **3 – Defined** | Tuned detections, documented processes | Custom rules, playbooks, defined IR process |
| **4 – Managed** | Metrics-driven, proactive | KPIs tracked, threat hunting, purple team exercises |
| **5 – Optimized** | Continuous improvement, automation | SOAR, ML-assisted detection, threat intel maturity |

### 10.2 90-Day Quick Start Milestones

| Week | Milestone |
|---|---|
| 1–2 | Complete asset inventory and log source audit |
| 3–4 | Select SIEM platform; begin Tier 1 log source onboarding |
| 5–6 | Enable OOTB detection rules; establish alert triage process |
| 7–8 | Onboard Tier 2 log sources; define escalation paths |
| 9–10 | Build first custom detection rules; deploy Tier 1 playbooks |
| 11–12 | First tabletop exercise; baseline KPIs established; 90-day retrospective |

### 10.3 Long-Term Roadmap (6–18 Months)

**6 Months:**
- All critical log sources onboarded
- MITRE ATT&CK coverage mapped and tracked
- Threat intelligence feeds integrated
- IR playbooks for top 5 scenarios complete

**12 Months:**
- First full purple team exercise completed
- Executive dashboard operational
- Compliance reporting automated or semi-automated
- Evaluate SOAR for alert automation and playbook orchestration

**18 Months:**
- SOAR implemented for top repetitive playbooks
- User and Entity Behavior Analytics (UEBA) integrated or enabled
- Threat hunting program initiated
- Annual program review and maturity assessment completed

### 10.4 Technology Expansion Considerations

As the program matures, evaluate adding:

| Technology | Value Added | Maturity Required |
|---|---|---|
| **SOAR** (e.g., Palo Alto XSOAR, Tines, Torq) | Automates repetitive response actions | Level 3+ |
| **UEBA** | Detects insider threats and anomalous behavior | Level 3+ |
| **NDR** (Network Detection & Response) | Deep packet inspection and network anomaly detection | Level 3+ |
| **XDR** | Unified detection across endpoint, network, cloud | Level 3+ |
| **Threat Hunting Platform** | Proactive hunt for undetected threats | Level 4+ |
| **Deception Technology** (honeypots, honeytokens) | High-confidence detection with near-zero false positives | Level 3+ |

---

## 📚 Additional Resources

### Frameworks & Standards
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [CIS Controls v8](https://www.cisecurity.org/controls/v8)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [CISA Free Cybersecurity Services & Tools](https://www.cisa.gov/resources-tools/resources/free-cybersecurity-services-and-tools)

### Communities & Threat Intelligence
- [MS-ISAC (Free for SLTT)](https://www.cisecurity.org/ms-isac)
- [AlienVault OTX](https://otx.alienvault.com/)
- [MISP Threat Sharing](https://www.misp-project.org/)
- [Sigma Rules (Community Detection Rules)](https://github.com/SigmaHQ/sigma)

### Training
- [SANS SEC555: SIEM with Tactical Analytics](https://www.sans.org/cyber-security-courses/siem-with-tactical-analytics/)
- [Elastic for Security — Free Training](https://www.elastic.co/training/free)
- [Microsoft Sentinel Ninja Training](https://techcommunity.microsoft.com/t5/microsoft-sentinel-blog/become-a-microsoft-sentinel-ninja-the-complete-level-400/ba-p/1246310)

---

## 🤝 Contributing

Contributions are welcome! If you have resources, playbooks, detection rules, or lessons learned from real SMB deployments, please open a pull request or issue.

---

## ⚠️ Disclaimer

This framework is provided for educational and guidance purposes. Every environment is different. Adapt these recommendations to your specific threat profile, regulatory requirements, and organizational constraints. This is not a substitute for qualified security advisory services.

---

*Last updated: 2025 | Maintained by the community*
