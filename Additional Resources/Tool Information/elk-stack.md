# ELK Stack Quickstart & Reference Guide
### Building Visibility in SMB Environments

---

## Table of Contents

1. [What is the ELK Stack?](#what-is-the-elk-stack)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [Installation](#installation)
   - [Elasticsearch](#elasticsearch)
   - [Logstash](#logstash)
   - [Kibana](#kibana)
   - [Beats Agents](#beats-agents)
5. [Basic Configuration](#basic-configuration)
6. [Ingesting Your First Logs](#ingesting-your-first-logs)
7. [Kibana Dashboards & Queries](#kibana-dashboards--queries)
8. [ELK vs Wazuh: When to Use Which](#elk-vs-wazuh-when-to-use-which)
9. [Hardening & Security Considerations](#hardening--security-considerations)
10. [Common Troubleshooting](#common-troubleshooting)
11. [Quick Reference Cheatsheet](#quick-reference-cheatsheet)

---

## What is the ELK Stack?

The **ELK Stack** is a collection of three open-source tools from Elastic that together form a powerful log management, search, and visualization platform:

| Component | Role |
|-----------|------|
| **E**lasticsearch | Distributed search and analytics engine; stores and indexes your data |
| **L**ogstash | Data processing pipeline; ingests, transforms, and ships data |
| **K**ibana | Web UI for searching, visualizing, and dashboarding Elasticsearch data |

A fourth component is commonly added:

| Component | Role |
|-----------|------|
| **Beats** | Lightweight data shippers installed on endpoints (Filebeat, Winlogbeat, Metricbeat, Packetbeat, etc.) |

This full stack is often called the **Elastic Stack** or **ELK + Beats**. In SMB environments, Beats agents on your endpoints ship logs directly to either Logstash (for enrichment) or Elasticsearch (for simpler setups).

---

## Architecture Overview

### Simple (Small SMB / Lab)
```
[Endpoints]
  Filebeat / Winlogbeat
       |
       v
[Elasticsearch + Kibana]  <-- single server or VM
```

### Recommended (SMB Production)
```
[Endpoints]
  Filebeat / Winlogbeat / Metricbeat
       |
       v
[Logstash]  <-- parsing, enrichment, filtering
       |
       v
[Elasticsearch Cluster]  <-- 1-3 nodes
       |
       v
[Kibana]  <-- dashboards, alerting, SIEM
```

> **SMB Note:** For environments under ~500 endpoints, a single Elasticsearch node with Kibana co-located is usually sufficient. Add Logstash only when you need custom parsing or multi-source enrichment.

---

## Prerequisites

- **OS:** Ubuntu 22.04 LTS or RHEL/Rocky 8/9 recommended
- **RAM:** Minimum 8 GB for a combined ES + Kibana server (16 GB preferred)
- **CPU:** 4 vCPUs minimum
- **Disk:** SSD strongly recommended; plan ~50–100 GB per 30 days of logs for a small environment
- **Java:** Elasticsearch bundles its own JDK — no separate install needed
- **Network:** Ensure TCP 9200, 9300 (Elasticsearch), 5601 (Kibana), and 5044 (Logstash Beats input) are accessible as needed

---

## Installation

All steps below assume **Ubuntu 22.04**. Commands require `sudo` or a root shell.

### Add the Elastic APT Repository

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt update
```

---

### Elasticsearch

```bash
sudo apt install elasticsearch -y
```

**Minimal `/etc/elasticsearch/elasticsearch.yml` for a single-node setup:**

```yaml
cluster.name: smb-siem
node.name: node-1
network.host: 0.0.0.0        # Change to specific IP in production
http.port: 9200
discovery.type: single-node  # Remove for multi-node clusters
xpack.security.enabled: true
xpack.security.http.ssl.enabled: false   # Enable in production with certs
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now elasticsearch

# Verify
curl -u elastic:<password> http://localhost:9200
```

On first start, Elasticsearch auto-generates a password for the `elastic` superuser. Capture it from the install output or reset it:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

---

### Logstash

```bash
sudo apt install logstash -y
```

**Example pipeline `/etc/logstash/conf.d/beats-input.conf`:**

```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  if [event][dataset] == "system.syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
    }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    user  => "elastic"
    password => "CHANGE_ME"
    index => "logs-%{[agent][type]}-%{+YYYY.MM.dd}"
  }
}
```

```bash
sudo systemctl enable --now logstash

# Test config
sudo /usr/share/logstash/bin/logstash --config.test_and_exit -f /etc/logstash/conf.d/
```

---

### Kibana

```bash
sudo apt install kibana -y
```

**`/etc/kibana/kibana.yml` essentials:**

```yaml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "CHANGE_ME"
```

Set the kibana_system password:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system
```

```bash
sudo systemctl enable --now kibana
```

Access Kibana at `http://<server-ip>:5601` and complete the enrollment wizard.

---

### Beats Agents

#### Filebeat (Linux / log files)

```bash
sudo apt install filebeat -y
```

**`/etc/filebeat/filebeat.yml` minimal config:**

```yaml
filebeat.inputs:
  - type: filestream
    id: syslog
    enabled: true
    paths:
      - /var/log/syslog
      - /var/log/auth.log

output.logstash:
  hosts: ["<logstash-ip>:5044"]

# OR direct to Elasticsearch (skip Logstash):
# output.elasticsearch:
#   hosts: ["http://<es-ip>:9200"]
#   username: "elastic"
#   password: "CHANGE_ME"
```

Enable built-in modules for common log sources:

```bash
sudo filebeat modules enable system nginx auditd
sudo filebeat setup
sudo systemctl enable --now filebeat
```

---

#### Winlogbeat (Windows Event Logs)

Download the Winlogbeat ZIP from [elastic.co/downloads](https://www.elastic.co/downloads/beats/winlogbeat), extract to `C:\Program Files\Winlogbeat`.

**`winlogbeat.yml` essentials:**

```yaml
winlogbeat.event_logs:
  - name: Application
    ignore_older: 72h
  - name: System
  - name: Security
  - name: Microsoft-Windows-Sysmon/Operational  # If Sysmon is installed (highly recommended)
  - name: Microsoft-Windows-PowerShell/Operational

output.logstash:
  hosts: ["<logstash-ip>:5044"]
```

```powershell
# Run as Administrator
cd "C:\Program Files\Winlogbeat"
.\install-service-winlogbeat.ps1
Start-Service winlogbeat
```

---

## Basic Configuration

### Index Lifecycle Management (ILM)

Prevent your disk from filling up by setting log retention policies in Kibana:

**Stack Management → Index Lifecycle Policies → Create Policy**

Recommended phases for SMB:

| Phase | Trigger | Action |
|-------|---------|--------|
| Hot | Default | Active indexing |
| Warm | After 7 days | Shrink shards, optimize |
| Cold | After 30 days | Read-only |
| Delete | After 90 days | Delete index |

Adjust retention based on your compliance requirements (PCI-DSS, HIPAA, etc. often require 1 year+).

---

### Index Templates

Apply standard mappings so your fields are correctly typed:

```bash
curl -u elastic:<password> -X PUT "http://localhost:9200/_index_template/logs-template" \
  -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0
    }
  }
}'
```

---

## Ingesting Your First Logs

After Filebeat or Winlogbeat is running, confirm data is flowing:

```bash
# Check Filebeat registry / status
sudo filebeat status

# Verify data in Elasticsearch
curl -u elastic:<password> "http://localhost:9200/logs-*/_count"
```

In Kibana:

1. Go to **Discover** (compass icon in the left sidebar)
2. Create a **Data View** matching `logs-*`
3. Set the time range to "Last 15 minutes"
4. You should see log events appear in the timeline

---

## Kibana Dashboards & Queries

### KQL (Kibana Query Language) Basics

| Goal | Query |
|------|-------|
| Filter by hostname | `host.name: "webserver01"` |
| Find failed SSH logins | `event.outcome: "failure" and process.name: "sshd"` |
| Windows logon failures | `event.code: 4625` |
| PowerShell execution | `process.name: "powershell.exe"` |
| Specific IP | `source.ip: "192.168.1.100"` |
| Multiple values | `event.code: (4624 or 4625 or 4648)` |
| Wildcard | `file.path: /etc/pass*` |

### Key Windows Event IDs to Monitor

| Event ID | Meaning | Priority |
|----------|---------|----------|
| 4624 | Successful logon | Medium |
| 4625 | Failed logon | High |
| 4648 | Logon with explicit credentials | High |
| 4672 | Special privileges assigned | High |
| 4688 | Process creation | High (with Sysmon) |
| 4698/4702 | Scheduled task created/modified | High |
| 4720 | User account created | High |
| 4732 | User added to privileged group | Critical |
| 7045 | New service installed | High |

### Useful Pre-built Dashboards

Kibana includes pre-built dashboards via **Integrations**. Recommended for SMB:

- **[Filebeat System]** – Linux syslog, auth.log
- **[Winlogbeat]** – Windows Security events
- **[Elastic Security]** – SIEM overview (requires Elastic Security app)
- **[Auditbeat System]** – File integrity, process, socket monitoring

Enable the Elastic Security app under **Stack Management → Kibana → Features** for a SIEM-style interface with detection rules.

---

## ELK vs Wazuh: When to Use Which

| Capability | ELK Stack | Wazuh |
|------------|-----------|-------|
| Log ingestion & search | Excellent | Good |
| Visualization / Dashboards | Excellent (Kibana) | Good (Kibana-based) |
| HIDS / Rootkit detection | ❌ No | ✅ Built-in |
| File Integrity Monitoring | Via Auditbeat | ✅ Built-in |
| Vulnerability assessment | Via integrations | ✅ Built-in |
| Compliance (PCI, HIPAA) | Via Elastic Security | ✅ Built-in modules |
| Active Response (auto-block) | ❌ No | ✅ Built-in |
| Out-of-box detection rules | Good (Elastic rules) | Excellent (OSSEC rules) |
| Complexity | Moderate | Lower for HIDS features |

**Recommendation for SMBs:** Wazuh excels as an endpoint detection and response (EDR)-adjacent tool. ELK excels at centralized log aggregation, custom dashboards, and scaling to diverse data sources. Many organizations run **both**: Wazuh agents shipping alerts into Elasticsearch for unified visibility.

---

## Hardening & Security Considerations

- **Enable TLS** for Elasticsearch HTTP and transport layers in production — never expose port 9200 to the internet
- **Create dedicated service accounts** — don't use the `elastic` superuser for Beats/Logstash output
- **Restrict Kibana** behind a VPN or reverse proxy (nginx) with authentication
- **Use API keys** instead of username/password in Beats configs where possible
- **Enable audit logging** in Elasticsearch to track who queries what
- **Set `xpack.security.enabled: true`** — it is the default in ES 8.x but verify it has not been disabled
- **Firewall rules:** Only allow Beats traffic (5044) from known agent subnets; never expose 9200/9300 externally

---

## Common Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Kibana "Unable to connect to Elasticsearch" | Wrong password or ES not running | Check ES status: `systemctl status elasticsearch` |
| Beats not sending data | Firewall blocking 5044 or wrong output config | `sudo filebeat test output` |
| No data in Discover | Wrong index pattern or time range | Adjust Data View and time picker |
| Logstash pipeline errors | Grok pattern mismatch | Use **Grok Debugger** in Kibana Dev Tools |
| Elasticsearch red/yellow status | Unassigned shards | Check `GET /_cluster/health` and `GET /_cat/shards?v` |
| High disk usage | Too many indices / no ILM | Apply ILM policy; manually delete old indices |
| Slow Kibana queries | Large time range + no filters | Narrow time range; add host/index filters |

---

## Quick Reference Cheatsheet

```bash
# Service management
sudo systemctl status elasticsearch logstash kibana filebeat

# Check Elasticsearch cluster health
curl -u elastic:<password> http://localhost:9200/_cluster/health?pretty

# List indices
curl -u elastic:<password> http://localhost:9200/_cat/indices?v

# Delete old index (careful!)
curl -u elastic:<password> -X DELETE http://localhost:9200/logs-filebeat-2024.01.01

# Test Filebeat config
sudo filebeat test config -e

# Test Filebeat output connectivity
sudo filebeat test output

# Tail Logstash logs
sudo journalctl -u logstash -f

# Reload Logstash pipeline without restart
curl -X POST http://localhost:9600/_node/reload

# Create Elasticsearch API key (for Beats)
curl -u elastic:<password> -X POST "http://localhost:9200/_security/api_key" \
  -H 'Content-Type: application/json' \
  -d '{"name":"filebeat-key","role_descriptors":{"filebeat_writer":{"cluster":["monitor","read_ilm"],"index":[{"names":["logs-*"],"privileges":["view_index_metadata","create_doc"]}]}}}'
```

---

## Additional Resources

- [Elastic Documentation](https://www.elastic.co/guide/index.html)
- [Elastic Security Detection Rules (GitHub)](https://github.com/elastic/detection-rules)
- [Filebeat Module Reference](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-modules.html)
- [MITRE ATT&CK Coverage in Elastic](https://www.elastic.co/security-labs/elastic-security-intelligence-analytics)
- [Sysmon Configuration (SwiftOnSecurity)](https://github.com/SwiftOnSecurity/sysmon-config) — strongly recommended for Windows endpoints

---

*Part of the SMB Visibility Resource Series | Companion guides: Wazuh Quickstart, Cost Calculator, Log Collection Policy*
