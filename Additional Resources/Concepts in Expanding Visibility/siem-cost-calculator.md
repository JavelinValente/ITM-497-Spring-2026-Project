# SMB Visibility Platform — Cost Calculation Guide
### Estimating Total Cost of Ownership for ELK Stack & Wazuh Deployments

---

## Table of Contents

1. [Overview: What You're Actually Paying For](#overview-what-youre-actually-paying-for)
2. [Hosting Costs](#hosting-costs)
   - [On-Premises](#on-premises)
   - [Cloud (IaaS)](#cloud-iaas)
   - [Managed / SaaS Options](#managed--saas-options)
3. [Software & Licensing Costs](#software--licensing-costs)
4. [Maintenance Costs](#maintenance-costs)
5. [Overhead & Hidden Costs](#overhead--hidden-costs)
6. [Cost Estimation Worksheets](#cost-estimation-worksheets)
   - [Small SMB (1–50 Endpoints)](#small-smb-150-endpoints)
   - [Mid SMB (51–250 Endpoints)](#mid-smb-51250-endpoints)
   - [Larger SMB (251–500 Endpoints)](#larger-smb-251500-endpoints)
7. [Cost Reduction Strategies](#cost-reduction-strategies)
8. [Build vs Buy Decision Framework](#build-vs-buy-decision-framework)

---

## Overview: What You're Actually Paying For

Total cost of ownership (TCO) for a visibility platform has four buckets that most budget estimates miss:

```
TCO = Hosting + Software/Licensing + Maintenance Labor + Overhead
```

Many organizations price only the hardware or cloud instance and are surprised by the true cost. This guide breaks down each component so you can build an honest budget before committing.

---

## Hosting Costs

### On-Premises

On-premises hosting requires upfront capital expenditure (CapEx) for hardware, plus ongoing operational expenditure (OpEx) for power, cooling, and rack space.

**Hardware sizing guidelines:**

| Environment | Endpoints | Recommended Server Spec | Estimated Hardware Cost |
|-------------|-----------|------------------------|------------------------|
| Lab / Micro | 1–25 | 8 GB RAM, 4 vCPU, 500 GB SSD | $400–$800 (refurb/mini PC) |
| Small SMB | 26–100 | 16 GB RAM, 8 vCPU, 2 TB SSD | $1,200–$2,500 |
| Mid SMB | 101–250 | 32 GB RAM, 16 vCPU, 4 TB SSD | $3,000–$6,000 |
| Larger SMB | 251–500 | 64 GB RAM, 32 vCPU, 8 TB SSD (RAID) | $6,000–$15,000 |

> **Important:** These are approximate ranges for refurbished or mid-tier server hardware. New enterprise hardware from Dell/HP can be 2–4x higher. Always get at least two quotes.

**Ongoing on-prem operational costs (annual estimates):**

| Cost Category | Annual Estimate | Notes |
|---------------|----------------|-------|
| Power (server) | $200–$600/yr | Based on 200–400W server, $0.12/kWh |
| Cooling overhead | $50–$150/yr | Approximate % of power cost |
| Rack space / colocation | $0 (own closet) to $2,400+/yr | If colocating at a data center |
| UPS / battery replacement | $50–$150/yr amortized | 3–5 year UPS battery life |
| Hardware refresh (amortized) | 20–25% of hardware cost/yr | Plan to replace every 4–5 years |

**On-prem total 3-year cost example (Mid SMB):**

| Item | Cost |
|------|------|
| Initial hardware | $4,500 |
| Power (3 years) | $1,200 |
| Cooling (3 years) | $360 |
| Hardware refresh reserve | $2,700 |
| **3-Year Total** | **~$8,760** |
| **Monthly Equivalent** | **~$244/mo** |

---

### Cloud (IaaS)

Cloud hosting converts CapEx to pure OpEx and allows you to scale storage and compute independently.

**Reference instance types (approximate monthly cost, on-demand):**

| Provider | Instance | vCPU | RAM | Approx. Cost/mo |
|----------|----------|------|-----|-----------------|
| AWS | t3.large | 2 | 8 GB | ~$60 |
| AWS | m6i.xlarge | 4 | 16 GB | ~$140 |
| AWS | m6i.2xlarge | 8 | 32 GB | ~$280 |
| Azure | D4s v5 | 4 | 16 GB | ~$140 |
| GCP | e2-standard-4 | 4 | 16 GB | ~$97 |
| Hetzner (EU/US) | CX32 | 4 | 8 GB | ~$15 |
| DigitalOcean | 4 vCPU / 8 GB | 4 | 8 GB | ~$48 |

> **Budget tip:** Reserved instances (1–3 year commitments) on AWS/Azure/GCP typically save 30–60% vs on-demand. Hetzner and DigitalOcean are significantly cheaper for SMBs that don't need major cloud integrations.

**Storage costs:**

Log data storage is often the largest ongoing cloud cost. Plan carefully.

| Storage Type | AWS (us-east-1) | Azure | Notes |
|-------------|-----------------|-------|-------|
| gp3 SSD | ~$0.08/GB/mo | ~$0.10/GB/mo | Hot storage (active search) |
| S3 / Blob | ~$0.023/GB/mo | ~$0.018/GB/mo | Cold archival |

**Estimating log volume:**

| Source | Approximate Daily Volume |
|--------|------------------------|
| Windows endpoint (Security log, moderate) | 50–200 MB/day |
| Linux server (syslog + auth) | 10–50 MB/day |
| Firewall (Palo Alto / Fortinet) | 100 MB–1 GB/day |
| Web server (nginx/Apache) | 50–500 MB/day |
| Network switch (syslog) | 5–30 MB/day |

> **Rule of thumb:** Budget approximately 1–5 GB/day per 50 endpoints for a typical SMB environment with standard Windows logging. After Elasticsearch compression, stored size is typically 30–50% of raw log size.

**Cloud monthly cost estimate (50 endpoints, 90-day retention):**

| Item | Monthly Cost |
|------|-------------|
| Compute (m6i.large, reserved 1yr) | ~$55 |
| Storage: ~150 GB/mo hot (gp3) | ~$12 |
| Storage: ~4.5 GB archived (S3) | ~$0.10 |
| Data transfer (egress) | ~$5–$20 |
| **Monthly Total** | **~$72–$90/mo** |

---

### Managed / SaaS Options

For SMBs that want visibility without self-managing infrastructure, several managed options exist:

| Option | Pricing Model | Approx. Cost | Notes |
|--------|--------------|-------------|-------|
| Elastic Cloud | GB of data / month | $95+/mo for small deployments | Fully managed Elasticsearch + Kibana |
| Elastic Cloud (Security) | Per endpoint | ~$10–$15/endpoint/mo | Adds Elastic Security (SIEM + EDR) |
| Wazuh Cloud | Per agent | ~$4/agent/mo (50 agents = $200/mo) | Managed Wazuh |
| Splunk Cloud | GB ingested/day | ~$150/GB/day (expensive at scale) | Powerful but costly for SMBs |
| Microsoft Sentinel | GB ingested | ~$2.46/GB/day (PAYG) | Good if already in Azure/M365 ecosystem |
| Datadog Security | Per host/mo | ~$23/host/mo | Better for infrastructure monitoring |

> **Comparison note:** A self-hosted ELK/Wazuh deployment typically runs 3–10x cheaper than a fully managed SaaS SIEM at SMB scale, but requires internal technical expertise. Use this guide to make an honest comparison factoring in your labor costs.

---

## Software & Licensing Costs

### Open Source (ELK Stack / Wazuh)

Both Wazuh and the core ELK Stack are open source and free to use:

| Software | License | Cost |
|----------|---------|------|
| Elasticsearch (Basic/Free tier) | Elastic License 2.0 | Free |
| Kibana (Basic/Free tier) | Elastic License 2.0 | Free |
| Logstash | Apache 2.0 | Free |
| Filebeat / Winlogbeat / Beats | Apache 2.0 | Free |
| Wazuh Manager, Agent, Dashboard | GPL v2 / Apache 2.0 | Free |

**Paid Elastic features (optional upgrades):**

| Feature | Tier | Monthly (estimated) |
|---------|------|-------------------|
| Machine learning anomaly detection | Platinum | ~$300+/mo (self-managed) |
| Advanced SIEM / alerting | Platinum | Included above |
| Elastic Endpoint Security (EDR) | Enterprise | Contact sales |
| Support SLA | Gold / Platinum | Varies |

For most SMBs, the free tier of Elasticsearch with Elastic Security provides sufficient capability without paid licensing.

### Supplementary Tools

| Tool | Purpose | Cost |
|------|---------|------|
| Sysmon (Microsoft) | Enhanced Windows telemetry | Free |
| Zeek (network sensor) | Network metadata | Free |
| Suricata (IDS/IPS) | Network detection | Free |
| NGINX (reverse proxy) | Securing Kibana | Free |
| Certbot / Let's Encrypt | TLS certificates | Free |

---

## Maintenance Costs

Maintenance is almost always the **largest long-term cost** and the one most frequently underestimated.

### Time Estimates by Task (Monthly)

| Task | Frequency | Hours/Month (Est.) |
|------|-----------|-------------------|
| Software updates (ES, Kibana, Beats) | Monthly | 1–2 hrs |
| OS patching (server) | Monthly | 0.5–1 hr |
| Review and tune detection rules/alerts | Weekly | 2–4 hrs |
| Investigate and triage alerts | Daily | 2–10 hrs |
| Index and storage management | Weekly | 0.5–1 hr |
| Onboard new log sources / agents | As needed | 1–3 hrs/source |
| Backup verification | Monthly | 0.5 hr |
| Documentation updates | Quarterly | 1–2 hrs |
| Incident response support | As needed | Highly variable |

**Monthly maintenance hour ranges by environment:**

| Environment | Passive Maintenance | Active Monitoring | Total Estimate |
|-------------|--------------------|--------------------|---------------|
| 1–50 endpoints | 3–5 hrs/mo | 5–15 hrs/mo | 8–20 hrs/mo |
| 51–250 endpoints | 5–10 hrs/mo | 10–30 hrs/mo | 15–40 hrs/mo |
| 251–500 endpoints | 8–15 hrs/mo | 20–50 hrs/mo | 28–65 hrs/mo |

### Labor Cost Calculation

```
Annual Maintenance Cost = Monthly Hours × 12 × Effective Hourly Rate
```

**Example rates (US market, 2024–2025):**

| Role | Fully Burdened Rate | Notes |
|------|--------------------|----|
| IT Generalist (internal) | $40–$75/hr | Fully burdened (salary + benefits + overhead) |
| Security Analyst (internal) | $65–$120/hr | Fully burdened |
| MSP / MSSP contract | $100–$200/hr | Outsourced management |
| Dedicated SOC analyst (FTE) | $80,000–$120,000/yr | Only warranted at larger scale |

**Example: Mid SMB, 100 endpoints, internal IT generalist:**

```
25 hrs/mo × 12 months × $60/hr = $18,000/year in maintenance labor
```

This single number often dwarfs all hosting and software costs combined.

---

## Overhead & Hidden Costs

### Training & Skill Development

| Activity | Estimated Cost |
|----------|---------------|
| Elastic Engineer training (official) | $1,500–$3,000/course |
| Elastic certifications (ECE, ECEA) | $400–$600/exam |
| Self-study resources, books, labs | $200–$500/yr |
| Conference / community events | $500–$2,000/yr |

A realistic budget for building internal ELK/SIEM competency: **$2,000–$5,000 one-time** per person, then **$500–$1,500/yr** for continuing education.

### Backup & Disaster Recovery

- Elasticsearch snapshots to S3/Blob: ~$0.023/GB/mo
- Recovery testing time: 2–4 hrs/quarter
- Backup storage for a 500 GB index: ~$11.50/mo on AWS S3

### Alert Fatigue & Tuning Overhead

Poor log collection policies (see the companion Log Collection Policy guide) lead to alert fatigue. Each noisy rule costs real analyst time. Budget 10–20% extra analyst time in the first 3–6 months for tuning.

### Compliance Overhead

If your organization has compliance requirements (PCI-DSS, HIPAA, SOC 2, CMMC), factor in:

| Activity | Cost |
|----------|------|
| Audit/assessment preparation | 10–40 hrs/yr of analyst time |
| Compliance-specific log retention (1–7 years) | Increased storage costs |
| Gap analysis and documentation | 5–20 hrs/yr |

### Network / Bandwidth

- Log forwarding from remote sites over WAN: may require VPN or Elastic Agent Fleet enrollment
- Beats agents generate low bandwidth (<1 Mbps for 50 endpoints) but consider WAN links at branch sites

---

## Cost Estimation Worksheets

### Small SMB (1–50 Endpoints)

| Category | Annual Cost (Self-Hosted) | Annual Cost (Managed SaaS) |
|----------|--------------------------|---------------------------|
| Hosting (cloud, Hetzner/DO) | $600–$1,200 | — |
| Managed platform (e.g., Elastic Cloud / Wazuh Cloud) | — | $1,500–$3,600 |
| Software licensing | $0 | Included above |
| Maintenance labor (8–20 hrs/mo @ $50/hr) | $4,800–$12,000 | $2,400–$4,800 (less mgmt) |
| Training (amortized, yr 1) | $1,000–$2,500 | $500–$1,000 |
| Backup / DR | $200–$400 | Included |
| **Annual Total** | **$6,600–$16,100** | **$4,400–$9,400** |

> **Takeaway:** At 1–50 endpoints, the managed SaaS option may be cheaper in year 1 when you account for training. Self-hosted becomes more economical in years 2–3 once expertise is established.

---

### Mid SMB (51–250 Endpoints)

| Category | Annual Cost (Self-Hosted) | Annual Cost (Managed SaaS) |
|----------|--------------------------|---------------------------|
| Hosting | $1,500–$4,000 | — |
| Managed platform | — | $6,000–$15,000 |
| Software licensing | $0 | Included |
| Maintenance labor (25–40 hrs/mo @ $60/hr) | $18,000–$28,800 | $12,000–$18,000 |
| Training | $2,000–$4,000 | $1,000–$2,000 |
| Backup / DR | $400–$800 | Included |
| **Annual Total** | **$21,900–$37,600** | **$19,000–$35,000** |

> **Takeaway:** At this scale, costs are comparable. The self-hosted route offers more customization and data control; managed SaaS offers faster time-to-value.

---

### Larger SMB (251–500 Endpoints)

| Category | Annual Cost (Self-Hosted) | Annual Cost (Managed SaaS) |
|----------|--------------------------|---------------------------|
| Hosting | $4,000–$10,000 | — |
| Managed platform | — | $15,000–$40,000 |
| Software licensing | $0–$5,000 | Included |
| Maintenance labor (50–65 hrs/mo @ $70/hr) | $42,000–$54,600 | $24,000–$36,000 |
| Training | $3,000–$6,000 | $1,500–$3,000 |
| Backup / DR | $800–$2,000 | Included |
| **Annual Total** | **$49,800–$77,600** | **$40,500–$79,000** |

> **Takeaway:** At this scale, a dedicated part-time security analyst becomes justifiable. Consider whether a fractional MSSP engagement (dedicated analyst hours) is more cost-effective than hiring.

---

## Cost Reduction Strategies

**1. Right-size your log collection** — Collecting everything is tempting but expensive. Focused log collection (see the companion Log Collection Policy guide) can cut storage costs by 40–70% while improving signal quality.

**2. Use ILM aggressively** — Move logs to warm/cold storage after 7 days. Archive to object storage (S3/Blob) for compliance retention at a fraction of hot storage costs.

**3. Leverage reserved/committed pricing** — If using AWS/Azure/GCP, 1-year reserved instances save 30–40%. Hetzner and Hive/OVH are viable for non-compliance environments.

**4. Automate routine maintenance** — Use cron jobs or Elastic Watcher to automate index cleanup, snapshot creation, and basic health checks. Saves 2–5 hrs/month.

**5. Implement detection rules before dashboards** — Dashboards are nice to have; automated alerting is need-to-have. Focus analyst time on building a lean, high-fidelity alert set first.

**6. Co-locate Elasticsearch and Kibana** — For environments under 200 endpoints, running both on a single server is fine and eliminates a second host cost.

**7. Tune before you scale** — Most SMB cost overruns come from storing noisy, low-value logs. Tuning costs $0 in software and pays dividends for years.

---

## Build vs Buy Decision Framework

Use this matrix to assess which direction fits your organization:

| Factor | Favor Self-Hosted | Favor Managed/SaaS |
|--------|------------------|--------------------|
| Budget | Lower ongoing costs (yr 2+) | Lower year-1 costs |
| Internal expertise | Have IT/security staff | Limited security expertise |
| Data control | Need on-prem data residency | Comfortable with cloud |
| Customization | Need custom log sources / parsers | Standard log sources |
| Time to value | Weeks to months | Days to weeks |
| Compliance | On-prem may be required | Cloud-compliant offerings available |
| Scale | Growing rapidly | Stable, small endpoint count |

**Quick rule of thumb:**

- Fewer than 50 endpoints + no internal security staff → **Consider managed SaaS or MSSP**
- 50–250 endpoints + at least one technical staff member → **Self-hosted ELK/Wazuh is viable**
- 250+ endpoints + dedicated IT staff → **Self-hosted with MSSP oversight is typically most cost-effective**

---

*Part of the SMB Visibility Resource Series | Companion guides: ELK Stack Quickstart, Wazuh Quickstart, Log Collection Policy*
