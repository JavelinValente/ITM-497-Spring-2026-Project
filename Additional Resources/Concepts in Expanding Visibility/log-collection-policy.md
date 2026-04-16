# Log Collection Policy Guide
### Determining What to Collect, What to Skip, and How to Capture Device Health Signals

---

## Table of Contents

1. [Why Log Policy Matters](#why-log-policy-matters)
2. [Core Principles of Log Collection](#core-principles-of-log-collection)
3. [What NOT to Collect (Low-Value / High-Noise Logs)](#what-not-to-collect-low-value--high-noise-logs)
4. [What to Collect (High-Value Logs)](#what-to-collect-high-value-logs)
   - [Windows Endpoints](#windows-endpoints)
   - [Linux / Unix Systems](#linux--unix-systems)
   - [Network Devices](#network-devices)
   - [Authentication Systems](#authentication-systems)
   - [Applications & Services](#applications--services)
5. [SNMP: Collecting Device Health Signals](#snmp-collecting-device-health-signals)
   - [What Is SNMP?](#what-is-snmp)
   - [Critical SNMP OIDs for SMBs](#critical-snmp-oids-for-smbs)
   - [Drive Failure Signals](#drive-failure-signals)
   - [System Overheating Signals](#system-overheating-signals)
   - [Power Failure Signals](#power-failure-signals)
   - [Network / Connectivity Signals](#network--connectivity-signals)
   - [SNMP Collection Tools](#snmp-collection-tools)
   - [Sending SNMP Traps to ELK / Wazuh](#sending-snmp-traps-to-elk--wazuh)
6. [Log Retention Policy](#log-retention-policy)
7. [Implementing the Policy](#implementing-the-policy)
8. [Tuning Over Time](#tuning-over-time)
9. [Quick Reference: Log Priority Matrix](#quick-reference-log-priority-matrix)

---

## Why Log Policy Matters

The instinct when standing up a SIEM is to log everything. In practice, this creates several problems:

- **Storage costs spiral** — unnecessary logs inflate index size 3–10x
- **Alert fatigue** — noisy, low-value events bury real threats
- **Slower queries** — searching across irrelevant data degrades analyst experience
- **Higher maintenance burden** — more data means more tuning, more false positives

A well-designed log policy answers three questions for every log source:

1. Does this log help us **detect threats**?
2. Does this log help us **investigate incidents**?
3. Does this log satisfy a **compliance requirement**?

If a log source answers "no" to all three, it probably shouldn't be collected.

---

## Core Principles of Log Collection

**Collect with intent, not by default.** Before enabling a new log source, define what detection or investigation scenario it serves.

**Prefer endpoint telemetry over raw log files.** Tools like Sysmon, Auditbeat, and Elastic Agent produce structured, enriched events that are far more useful than raw text logs.

**Normalize at collection time.** Use Logstash, Elastic Agent ingest pipelines, or Beats modules to parse and tag logs before they land in Elasticsearch. Unstructured logs are searchable but not correlated effectively.

**Log the "who, what, where, when."** Authentication events, process creation, file access, and network connections are the foundation of any investigation. Everything else is secondary.

**Start minimal, expand deliberately.** It's easier to add log sources than to remove them once analysts depend on them. Begin with the high-priority sources in this guide and add others based on detected gaps.

---

## What NOT to Collect (Low-Value / High-Noise Logs)

Disabling these saves significant storage and reduces noise without meaningful loss of visibility.

### Windows Event Logs to Exclude or Filter

| Event ID | Description | Why Skip |
|----------|------------|----------|
| 4656 | Handle to object requested | Extremely high volume; rarely actionable without full object access audit |
| 4658 | Handle closed | Volume is enormous; no investigative value alone |
| 4663 (bulk) | Object access — file read/write | Enable only for specific sensitive paths (e.g., HR, finance shares), not system-wide |
| 5152 / 5156 | Windows Filtering Platform packet drop/allow | Enormous volume from normal network activity |
| 4798 / 4799 | User/Group membership enumeration | High volume from legitimate tools; tune to specific accounts if needed |
| 5379 | Credential manager read | Very high volume; noisy |
| 4703 | Token rights adjustment | High frequency from normal system operations |
| 7036 | Service Control Manager state changes | Useful for specific services only; don't collect globally |

### Linux / Syslog Sources to Filter

| Source | Why Skip / Filter |
|--------|------------------|
| `/var/log/mail.log` | Only relevant if running a mail server; skip otherwise |
| `/var/log/kern.log` (bulk) | Useful for kernel panics and hardware errors; filter out routine kernel messages |
| cron job output (verbose) | Log cron job start/stop, not stdout of each job unless anomalous |
| `/var/log/lastlog` | Binary file; use `last` / `lastb` output instead, forwarded sparingly |
| Application debug logs | Never forward debug-level logs to SIEM; use local log rotation only |

### Network Device Logs to Filter

| Log Type | Why Skip |
|----------|---------|
| ICMP (ping) traffic logs | High volume; almost always noise |
| Broadcast / multicast traffic | Background noise; no investigative value |
| Known-good backup traffic (large transfers at scheduled times) | Tune out with time-of-day and source/dest IP filters |
| Normal DHCP leases (high-volume renewals) | Log only new assignments and conflicts, not every renewal |
| CDP / LLDP neighbor discovery | Useful periodically; not worth continuous streaming |

### Application Logs to Filter

| Log Type | Why Skip |
|----------|---------|
| Successful health check / load balancer probes | Hundreds per minute; no value |
| Verbose ORM / SQL debug logs | Only valuable in development environments |
| Static asset requests (images, CSS, JS) | Web server access logs: forward only non-200 and POST requests |
| Application heartbeat logs | Internal keep-alives from monitoring agents |

---

## What to Collect (High-Value Logs)

### Windows Endpoints

Enable **Sysmon** on all Windows endpoints before anything else. It provides process creation, network connections, file creation, and registry changes with far more detail than native Windows logging. Recommended config: [SwiftOnSecurity/sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config)

**Windows Event IDs — Collect and Alert:**

| Event ID | Source | Description | Priority |
|----------|--------|-------------|----------|
| 4624 | Security | Successful logon | Medium |
| 4625 | Security | Failed logon | High |
| 4648 | Security | Logon using explicit credentials (runas, lateral movement) | High |
| 4672 | Security | Special privileges assigned at logon | High |
| 4688 | Security | Process creation (enable command-line logging in GPO) | High |
| 4698 | Security | Scheduled task created | High |
| 4702 | Security | Scheduled task modified | High |
| 4720 | Security | User account created | High |
| 4722 | Security | User account enabled | Medium |
| 4723 / 4724 | Security | Password change / reset | Medium |
| 4726 | Security | User account deleted | High |
| 4732 | Security | Member added to privileged group | Critical |
| 4740 | Security | Account locked out | High |
| 4776 | Security | NTLM authentication attempt | Medium |
| 7045 | System | New service installed | High |
| 7034 / 7035 | System | Service crashed / status changed | Medium |
| 1102 | Security | Audit log cleared — potential attacker evasion | Critical |
| 4104 | PowerShell | Script block logging — PowerShell execution | High |
| 4688 + Sysmon 1 | Both | Process creation with full command line | High |

**Sysmon Events — Collect All, Alert on Subset:**

| Sysmon Event ID | Description | Alert Threshold |
|----------------|-------------|----------------|
| 1 | Process creation | Alert on suspicious parents/children |
| 3 | Network connection | Alert on non-standard processes making outbound connections |
| 7 | Image (DLL) loaded | Alert on suspicious DLL loads |
| 8 | CreateRemoteThread | Alert always (injection technique) |
| 10 | Process access | Alert on LSASS access (credential theft) |
| 11 | File created | Alert on executables in temp/download paths |
| 12/13/14 | Registry events | Alert on persistence key modifications |
| 22 | DNS query | Baseline and alert on anomalous domains |

---

### Linux / Unix Systems

| Log Source | Path | What to Collect |
|------------|------|----------------|
| Authentication | `/var/log/auth.log` or `/var/log/secure` | All SSH logon success/failure, sudo usage, su attempts |
| Syslog | `/var/log/syslog` | System daemon messages, hardware errors |
| Audit daemon | `/var/log/audit/audit.log` | Configure auditd rules for: privileged command execution, file access on sensitive paths, user/group modifications |
| Kernel messages | `journalctl -k` | Hardware errors, OOM killer events, filesystem errors |
| Cron | `/var/log/cron` | Job execution (not output) |
| Application error logs | Varies | HTTP 4xx/5xx, application exceptions |

**Recommended auditd rules (add to `/etc/audit/rules.d/audit.rules`):**

```
# Privileged commands
-a always,exit -F path=/usr/bin/sudo -F perm=x -k privileged
-a always,exit -F path=/usr/bin/su -F perm=x -k privileged
-a always,exit -F path=/usr/bin/passwd -F perm=x -k privileged

# User and group modifications
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/sudoers -p wa -k identity

# Sensitive directories
-w /etc/ssh/sshd_config -p wa -k sshd_config
-w /root/ -p wa -k root_activity

# Scheduled tasks
-w /etc/cron.d/ -p wa -k cron
-w /var/spool/cron/ -p wa -k cron
```

---

### Network Devices

| Source | What to Collect | Priority |
|--------|----------------|----------|
| Firewall allow/deny logs | Denied traffic by external IPs, allowed traffic on sensitive ports | High |
| Firewall authentication | Admin login attempts, config changes | Critical |
| VPN authentication | Login success/failure, session duration | High |
| DNS logs | All queries (if internal resolver); flag non-existent domain responses (NXDOMAIN spike = possible C2 beaconing) | High |
| DHCP assignments | New device assignments, hostname changes | Medium |
| Switch/router config changes | Any config change events | Critical |
| Port security violations | Unauthorized device plugged in | High |
| BGP / routing changes | Route table changes (uncommon = suspicious) | High |

---

### Authentication Systems

| Source | What to Collect |
|--------|----------------|
| Active Directory / LDAP | All logon events, group membership changes, GPO changes, account lockouts, replication errors |
| Azure AD / Entra ID | Sign-in logs (success and failure), MFA events, Conditional Access policy matches/blocks, risky sign-in detections |
| Okta / Duo | All authentication events, MFA push denials, policy bypass attempts |
| RADIUS (Wi-Fi auth) | Authentication success/failure, device MAC addresses |

---

### Applications & Services

| Application | What to Log |
|-------------|------------|
| Web servers (nginx, Apache, IIS) | HTTP 400/401/403/404/500 errors, POST requests, non-standard user agents |
| Email (Exchange, Google Workspace) | Admin actions, mail forwarding rules created, unusual send volumes |
| Remote desktop / VPN | Session creation, source IPs, duration |
| File sharing (SharePoint, NAS) | Mass download events, permission changes on sensitive shares |
| Backup software | Job success/failure, unexpected size changes |
| Antivirus / EDR | Detection events, quarantine actions, scan failures |

---

## SNMP: Collecting Device Health Signals

### What Is SNMP?

**Simple Network Management Protocol (SNMP)** is a protocol used to monitor and manage network devices — switches, routers, firewalls, UPS units, servers, printers, and more. SNMP works in two modes:

- **Polling:** Your monitoring system queries a device for its current state (CPU temp, disk status, interface counters) on a schedule
- **Traps:** The device proactively sends an alert to your collector when a threshold is crossed or an event occurs (power failure, drive fault)

SNMP data complements your log data by capturing **hardware state** that syslog or event logs often miss. A drive failure, power surge, or thermal event may not generate a Windows/Linux log but will generate an SNMP trap.

**SNMP versions:**

| Version | Security | Use Case |
|---------|---------|---------|
| SNMPv1 | None (community string in plain text) | Legacy devices only; avoid |
| SNMPv2c | Community string (still plain text) | Most SMB devices today |
| SNMPv3 | Authentication + encryption | Recommended wherever supported |

---

### Critical SNMP OIDs for SMBs

OIDs (Object Identifiers) are the addresses for specific data points in SNMP. Below are the most important categories for SMB infrastructure monitoring.

**Standard MIBs (universal across vendors):**

| OID | MIB | Description |
|-----|-----|-------------|
| `1.3.6.1.2.1.1.3.0` | SNMPv2-MIB | System uptime |
| `1.3.6.1.2.1.2.2.1.8` | IF-MIB | Interface operational status |
| `1.3.6.1.2.1.2.2.1.10/16` | IF-MIB | Interface input/output octets |
| `1.3.6.1.4.1.2021.11` | UCD-SNMP-MIB | CPU load (Linux net-snmp) |
| `1.3.6.1.4.1.2021.4` | UCD-SNMP-MIB | Memory usage (Linux net-snmp) |
| `1.3.6.1.4.1.2021.9` | UCD-SNMP-MIB | Disk usage (Linux net-snmp) |
| `1.3.6.1.4.1.2021.13.15` | UCD-DISK-MIB | Disk error flags (Linux) |

---

### Drive Failure Signals

Drive failures are one of the most business-critical hardware events. Multiple SNMP paths provide early warning:

**Generic Linux SMART via net-snmp (UCD-DISK-MIB):**

| OID | Description | Alert On |
|-----|-------------|---------|
| `1.3.6.1.4.1.2021.9.1.100.1` | dskErrorFlag — disk 1 | Value = 1 (error) |
| `1.3.6.1.4.1.2021.9.1.2.1` | dskPath — mount point | Reference |
| `1.3.6.1.4.1.2021.9.1.9.1` | dskPercent — disk % used | Alert above 85–90% |

**RAID / Storage Controller (vendor-specific):**

| Vendor | MIB / OID Area | Drive Failure OID (example) |
|--------|----------------|---------------------------|
| Dell PowerEdge (iDRAC/OpenManage) | `1.3.6.1.4.1.674.10892.5` | `...physicalDiskState` — value 4 = failed |
| HP ProLiant (iLO / HPE Insight) | `1.3.6.1.4.1.232.3.2.3.1.1.11` | `cpqDaPhyDrvStatus` — 3 = failed |
| Synology NAS | `1.3.6.1.4.1.6574.2.1.1.5` | `diskStatus` — 5 = crashed |
| QNAP NAS | `1.3.6.1.4.1.24681.1.2.11.1.7` | `HdStatus` — failure codes vary |

**SNMP Trap to watch for (disk events):**

```
Trap OID: .1.3.6.1.4.1.674.10892.5.0.2048  (Dell: disk state change)
Trap OID: .1.3.6.1.4.1.232.0.3014          (HP: physical drive failed)
```

**Best practice:** Combine SNMP disk polling with Wazuh's file integrity monitoring and Elasticsearch cluster health checks for comprehensive storage visibility. SNMP catches hardware-layer failures; the SIEM catches filesystem-level anomalies.

---

### System Overheating Signals

Thermal events can precede hardware failure and in some cases indicate physical security issues (e.g., cooling tampering).

**Host Temperature (Linux via LM-Sensors / net-snmp extension):**

| OID | Description |
|-----|-------------|
| `1.3.6.1.4.1.2021.13.16.2.1.3.1` | LM-SENSORS-MIB — sensor value (temperature, milli-celsius) |
| `1.3.6.1.4.1.2021.13.16.2.1.2.1` | Sensor name (e.g., "Core 0") |

Alert if temperature exceeds 70–80°C for CPU cores or 40–50°C for hard drives.

**Vendor-Specific Temperature OIDs:**

| Device Type | OID Example | Alert Threshold |
|-------------|-------------|----------------|
| Dell PowerEdge | `1.3.6.1.4.1.674.10892.5.4.700.20.1.6` (inlet temp) | > 35°C inlet |
| HP ProLiant | `1.3.6.1.4.1.232.6.2.6.8.1.4` (cpqHeTemperatureCondition) | Value 4 = failed |
| Cisco switch (ambient) | `1.3.6.1.4.1.9.9.13.1.3.1.3` (ciscoEnvMonTemperatureValue) | > 45°C for switches |
| APC UPS battery temp | `1.3.6.1.4.1.318.1.1.1.2.2.2` | > 40°C |

**Thermal SNMP Traps:**

```
Trap OID: .1.3.6.1.4.1.674.10892.5.0.1163   (Dell: temperature exceeded threshold)
Trap OID: .1.3.6.1.4.1.232.0.6011           (HP: temperature out of range)
Trap OID: .1.3.6.1.4.1.9.9.13.3.0.1         (Cisco: temperature alarm)
```

---

### Power Failure Signals

Power events — UPS switchover, battery low, power restored — are critical for availability monitoring and can indicate physical tampering or environmental issues.

**APC / Schneider UPS (PowerNet-MIB) — most common in SMBs:**

| OID | Description | Alert Condition |
|-----|-------------|----------------|
| `1.3.6.1.4.1.318.1.1.1.4.1.1` | upsBasicOutputStatus | 2 = on battery, 3 = on smart boost |
| `1.3.6.1.4.1.318.1.1.1.2.2.1` | upsAdvBatteryCapacity | Alert < 20% |
| `1.3.6.1.4.1.318.1.1.1.2.2.4` | upsAdvBatteryReplaceIndicator | 2 = needs replacement |
| `1.3.6.1.4.1.318.1.1.1.3.2.1` | upsAdvInputLineVoltage | Alert on large deviations from nominal |
| `1.3.6.1.4.1.318.1.1.1.2.1.1` | upsBasicBatteryStatus | 2 = low battery |

**Critical APC SNMP Traps:**

| Trap OID | Meaning |
|----------|---------|
| `.1.3.6.1.4.1.318.0.5` | On battery — power failure detected |
| `.1.3.6.1.4.1.318.0.6` | Power restored |
| `.1.3.6.1.4.1.318.0.9` | Low battery — imminent shutdown |
| `.1.3.6.1.4.1.318.0.15` | Battery needs replacement |
| `.1.3.6.1.4.1.318.0.19` | UPS overloaded |

**Eaton / Tripp-Lite UPS:** OID structures differ; download the vendor MIB from the manufacturer and load it into your SNMP tool.

**PDU / Power Distribution Units:**

Rack PDUs often support SNMP for outlet-level monitoring. Alert on outlet current draw spikes (potential hardware failure drawing excess power) or unexpected outlet state changes.

---

### Network / Connectivity Signals

| Signal | OID | Alert Condition |
|--------|-----|----------------|
| Interface link down | `1.3.6.1.2.1.2.2.1.8` (ifOperStatus) | Value 2 = down |
| High interface error rate | `1.3.6.1.2.1.2.2.1.14` (ifInErrors) | Threshold: errors/min > 100 |
| Spanning tree topology change | Trap: `.1.3.6.1.6.3.1.1.5.x` | Any topology change trap |
| CPU utilization | `1.3.6.1.4.1.9.9.109.1.1.1.1.6` (Cisco) | > 90% for >5 min |
| Memory utilization | `1.3.6.1.4.1.9.9.48.1.1.1.5` (Cisco) | < 10% free |
| Fan failure | `1.3.6.1.4.1.9.9.13.1.4.1.3` (Cisco) | Value 5 = failed |

---

### SNMP Collection Tools

**For polling (active queries):**

| Tool | Description | Cost |
|------|-------------|------|
| **Zabbix** | Full-featured open-source monitoring with SNMP polling, templated alerting | Free |
| **Prometheus + SNMP Exporter** | SNMP polling that feeds Prometheus; pairs with Grafana | Free |
| **LibreNMS** | Network-focused; auto-discovers devices, built-in SNMP templates | Free |
| **PRTG** (up to 100 sensors) | Windows-based; easy setup; limited free tier | Free / Paid |
| **Nagios / Icinga** | Classic SNMP monitoring; steep learning curve | Free |

**For traps (device-initiated alerts):**

| Tool | Description |
|------|-------------|
| **snmptrapd** (net-snmp) | Standard Linux trap daemon; can forward to syslog or scripts |
| **Zabbix** | Built-in trap receiver |
| **Logstash snmp input plugin** | Receives SNMP traps and forwards to Elasticsearch |

---

### Sending SNMP Traps to ELK / Wazuh

**Option A: snmptrapd → syslog → Filebeat → Elasticsearch**

```ini
# /etc/snmp/snmptrapd.conf
authCommunity log,execute,net public
traphandle default /usr/sbin/snmptthandler
# Or simply log to syslog:
logOption s  # syslog output
```

Then configure Filebeat to forward `/var/log/syslog` where traps appear.

**Option B: Logstash SNMP input plugin (polling)**

```ruby
input {
  snmp {
    get => ["1.3.6.1.4.1.318.1.1.1.4.1.1", "1.3.6.1.4.1.318.1.1.1.2.2.1"]
    hosts => [{ host => "udp:192.168.1.20/161" community => "public" version => "2c" }]
    interval => 60
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "snmp-device-health-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "CHANGE_ME"
  }
}
```

**Option C: Wazuh SNMP trap integration**

Wazuh can consume SNMP traps via its syslog input. Configure `snmptrapd` to write to a file, then add a Wazuh localfile rule:

```xml
<!-- /var/ossec/etc/ossec.conf -->
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/snmptraps.log</location>
</localfile>
```

Create custom Wazuh decoder and rules for SNMP trap patterns to generate alerts.

---

## Log Retention Policy

Retention decisions should balance compliance requirements, investigative need, and storage cost.

| Log Category | Minimum Retention | Recommended | Compliance Notes |
|-------------|-------------------|-------------|-----------------|
| Authentication events (Windows/AD) | 90 days | 1 year | PCI-DSS requires 1 year; HIPAA requires 6 years |
| Firewall / network flow logs | 90 days | 1 year | PCI-DSS, NIST 800-53 |
| Endpoint process / Sysmon events | 30 days | 90 days | Investigative window |
| DNS query logs | 30 days | 90 days | Useful for C2 retrospective analysis |
| SNMP device health | 30 days | 90 days | Trending and incident correlation |
| Application logs (web, email) | 30 days | 1 year | Varies by application sensitivity |
| Backup / integrity logs | 1 year | 3 years | Audit trail for data protection |

**Storage tiers:**

- Hot (Elasticsearch searchable): 7–30 days depending on volume and budget
- Warm (Elasticsearch, reduced replicas): 30–90 days
- Cold / Archive (S3, Blob, or tape): 90 days to compliance maximum

---

## Implementing the Policy

### Step 1: Inventory Your Log Sources

Create a spreadsheet with every potential log source:

- Source name and IP
- Log type (Windows events, syslog, SNMP trap, etc.)
- Estimated daily volume
- Answer: Detect / Investigate / Compliance value

### Step 2: Enable High-Priority Sources First

In order of ROI for SMBs:

1. Windows Security log (domain controllers first, then all endpoints)
2. Sysmon on all Windows endpoints
3. Linux `auth.log` / auditd on all servers
4. Firewall authentication and deny logs
5. SNMP traps from UPS, servers, and core network devices
6. DNS logs from internal resolver
7. VPN / remote access authentication
8. Active Directory domain controller events

### Step 3: Configure Filtering at the Source or Collector

Filter before data reaches Elasticsearch whenever possible — filtering in Logstash is far cheaper than storing and then deleting.

```ruby
# Logstash filter example: drop high-volume, low-value events
filter {
  if [winlog][event_id] in [4656, 4658, 5152, 5156, 4799] {
    drop {}
  }
  # Drop successful health check probes
  if [http][response][status_code] == 200 and [url][path] =~ /^\/health/ {
    drop {}
  }
}
```

### Step 4: Document Your Policy

Maintain a log collection policy document that includes:

- Which sources are collected and why
- Which sources are explicitly excluded and why
- Retention period for each category
- Owner responsible for each source
- Last review date

Review and update the policy quarterly or after any significant incident.

---

## Tuning Over Time

A log policy is never "set and forget." After initial deployment:

**Week 1–2:** Identify the top 5 noisiest sources. Are they generating actionable alerts? If not, consider filtering or reducing verbosity.

**Month 1:** Review alert volume. If more than 50 alerts/day require manual triage, the policy needs tuning. Use the Kibana **Alerts** dashboard to identify which detection rules fire most frequently and assess their true positive rate.

**Month 2–3:** Run a **data volume report** in Kibana (Stack Management → Index Management) to see which indices consume the most storage. Cross-reference against alert value.

**Quarterly:** Review the log source inventory. Add sources that address identified visibility gaps; remove or filter sources that produced no actionable detections.

**After any incident:** Ask what log data you *wished* you had during the investigation and add those sources. Ask what log data you had to wade through that added no value and filter it.

---

## Quick Reference: Log Priority Matrix

| Log Source | Collect? | Alert? | Priority |
|------------|----------|--------|----------|
| Windows Security (4624/4625/4672) | ✅ Yes | ✅ Yes | Critical |
| Windows Security (4656/4658/5152) | ❌ Filter | ❌ No | Noise |
| Sysmon process creation (Event 1) | ✅ Yes | ✅ Selective | High |
| Sysmon LSASS access (Event 10) | ✅ Yes | ✅ Always | Critical |
| PowerShell script block (Event 4104) | ✅ Yes | ✅ Yes | High |
| Linux auth.log / SSH | ✅ Yes | ✅ Yes | High |
| Linux auditd (sensitive paths) | ✅ Yes | ✅ Yes | High |
| Linux kernel (hardware errors) | ✅ Yes | ✅ Yes | High |
| Linux app debug logs | ❌ No | ❌ No | Noise |
| Firewall deny logs (external) | ✅ Yes | ✅ Selective | High |
| Firewall admin auth | ✅ Yes | ✅ Always | Critical |
| ICMP / broadcast traffic logs | ❌ Filter | ❌ No | Noise |
| DNS query logs | ✅ Yes | ✅ On anomaly | High |
| DHCP (new assignments) | ✅ Yes | ✅ New devices | Medium |
| SNMP — UPS power events | ✅ Yes | ✅ Always | Critical |
| SNMP — Drive failure traps | ✅ Yes | ✅ Always | Critical |
| SNMP — Temperature threshold | ✅ Yes | ✅ Always | High |
| SNMP — Interface link down | ✅ Yes | ✅ Yes | High |
| SNMP — Routine interface counters | ✅ Yes (trending) | ❌ Threshold only | Low |
| AD Group membership changes | ✅ Yes | ✅ Privileged groups | Critical |
| Backup job failure | ✅ Yes | ✅ Always | High |
| Web server 200 static assets | ❌ Filter | ❌ No | Noise |
| Web server 400/403/500 errors | ✅ Yes | ✅ Spike detection | High |
| Load balancer health check hits | ❌ Filter | ❌ No | Noise |

---

*Part of the SMB Visibility Resource Series | Companion guides: ELK Stack Quickstart, Wazuh Quickstart, Cost Calculator*
