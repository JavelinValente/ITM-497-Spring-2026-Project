# SNMP Trapping & Polling for Organizational Visibility

> **Framework Section:** Log Source Strategy & Expanding Visibility
> **Audience:** SMB IT Administrators, Network Engineers, Security Teams
> **Skill Level:** Beginner to Intermediate

---

## Overview

Simple Network Management Protocol (SNMP) is a foundational method for monitoring and managing network devices in an SMB environment. By leveraging SNMP Trapping and Polling, organizations gain visibility into infrastructure that does not support traditional endpoint agents — such as switches, routers, firewalls, printers, and UPS systems.

SNMP data can be ingested into a SIEM (such as Wazuh) to correlate network-level events with endpoint telemetry, providing a more complete operational picture across the environment.

> **Why this matters for SMBs:** Many SMB environments have rich network infrastructure but limited visibility into it. SNMP is often already enabled on devices — it just needs to be pointed at a collector and forwarded into your SIEM.

---

## 1. SNMP Trapping vs. Polling Explained

There are two primary methods for collecting SNMP data. Most mature environments use both in combination.

| | SNMP Polling | SNMP Trapping |
|---|---|---|
| **How it works** | Monitor actively queries devices on a schedule | Devices send alerts to a trap receiver when an event occurs |
| **Direction** | Pull (monitor → device) | Push (device → monitor) |
| **Frequency** | Configurable interval (e.g., every 60 seconds) | Event-driven (immediate) |
| **Latency** | Delayed by poll interval | Near real-time |
| **Network load** | Moderate (regular queries across all devices) | Low (only on events) |
| **Data richness** | Full MIB data available on each poll | Limited to OIDs defined in the trap |
| **Missed events** | Possible if event resolves between polls | Possible if device is unreachable when event fires |
| **Best for** | Performance baselines, dashboards, capacity planning | Fault detection, security alerts, immediate notification |

### When to Use Each

- **Use Polling for:** Continuous monitoring of CPU/memory/interface utilization, uptime, bandwidth trends, and capacity planning.
- **Use Trapping for:** Detecting link-down events, authentication failures, config changes, hardware faults, and UPS battery alerts in real time.
- **Best practice:** Use **both** — polling establishes baselines and dashboards, trapping provides immediate alerting on critical events.

---

## 2. SMB-Relevant Devices

SNMP is supported on a wide range of devices commonly found in SMB environments. Enabling SNMP on these devices significantly expands coverage beyond endpoint agents.

| Device Type | Key Data Points | Example Vendors |
|---|---|---|
| **Managed Switches** | Port status, MAC address table, VLAN info, error counters, STP state | Cisco, Netgear, HP/Aruba, Ubiquiti |
| **Routers / Firewalls** | Interface traffic, routing table, CPU/memory, auth failures, VPN tunnels | Cisco, Fortinet, pfSense, Meraki |
| **Wireless APs** | Client associations, signal strength, rogue AP detection | Ubiquiti UniFi, Cisco Meraki, Aruba |
| **UPS Systems** | Battery level, load %, runtime remaining, on-battery events, bypass alerts | APC, Eaton, Vertiv |
| **Printers / MFDs** | Toner levels, error states, job counts, supply alerts | HP, Xerox, Ricoh, Kyocera |
| **Servers (IPMI/iDRAC)** | Fan speed, temperature sensors, PSU status, hardware faults | Dell iDRAC, HP iLO, Supermicro BMC |
| **NAS / Storage** | Disk health, RAID status, volume utilization, read/write errors | Synology, QNAP, TrueNAS |
| **Environmental Sensors** | Temperature, humidity, door/rack intrusion | Raritan, APC NetBotz, Geist |

### What to Prioritize First

For most SMBs, start with the highest-impact sources:

1. **Core network switches and routers** — authentication failures and link changes are high-value security signals
2. **Firewall / perimeter devices** — interface status and auth failures
3. **UPS systems** — power events can be early indicators of physical security issues or outages
4. **Servers with IPMI/iDRAC** — hardware faults before they cause downtime

---

## 3. Integrating SNMP with Wazuh / SIEM

Wazuh does not natively receive SNMP traps directly, but SNMP data can be forwarded into the Wazuh pipeline using intermediate tools. The most common approach is to use an SNMP daemon or Network Management System (NMS) to convert SNMP data into syslog, which Wazuh ingests and parses.

### Recommended Integration Architecture

```
[ Network Devices ]
       |
       | SNMP Traps (UDP 162) / Polling (UDP 161)
       v
[ NMS / Trap Receiver ]          <-- LibreNMS, Zabbix, snmptrapd, PRTG
       |
       | Syslog forwarding (UDP/TCP 514)
       v
[ Wazuh Server ]
       |
       | Parsed & decoded events
       v
[ Wazuh Indexer → Dashboard ]    <-- Alerts, dashboards, compliance
```

### Step-by-Step Setup

1. **Configure SNMP on target devices** — enable SNMP, set community strings or v3 credentials, point trap destination to your NMS server IP
2. **Deploy an NMS / trap receiver** — install LibreNMS, Zabbix, or snmptrapd on a dedicated server or VM
3. **Configure syslog forwarding** — set NMS to forward SNMP events as syslog messages to the Wazuh Server (port 514)
4. **Create Wazuh decoders and rules** — write or import decoders to parse SNMP syslog messages; create detection rules for key events (link-down, auth failure, etc.)
5. **Build dashboards and alerts** — surface network-level events in the Wazuh Dashboard alongside endpoint telemetry

### NMS Tool Comparison

| Tool | Role | Cost | Notes |
|---|---|---|---|
| **LibreNMS** | Full NMS — polling, trapping, alerting, syslog forwarding | Free / Open-source | Best all-in-one for SMBs; auto-discovers devices |
| **Zabbix** | SNMP polling + trap receiver with syslog export | Free / Open-source | Highly scalable; steeper learning curve |
| **PRTG Network Monitor** | Easy SNMP monitoring with built-in dashboards | Free up to 100 sensors | Good UI; commercial above 100 sensors |
| **snmptrapd (Net-SNMP)** | Lightweight standalone trap receiver | Free / Open-source | Simple; pairs well with rsyslog for forwarding |
| **Checkmk** | SNMP monitoring with alerting and integrations | Free (Raw Edition) | Good balance of ease and power |

### Key Resources

- [Wazuh Syslog Remote Service Configuration](https://documentation.wazuh.com/current/user-manual/manager/remote-service.html)
- [Wazuh Custom Decoders](https://documentation.wazuh.com/current/user-manual/ruleset/custom-decoders.html)
- [LibreNMS SNMP Configuration Examples](https://docs.librenms.org/Support/SNMP-Configuration-Examples/)
- [LibreNMS Syslog Integration](https://docs.librenms.org/Extensions/Syslog/)
- [Zabbix SNMP Monitoring Documentation](https://www.zabbix.com/documentation/current/en/manual/config/items/itemtypes/snmp)
- [Net-SNMP snmptrapd Man Page](http://www.net-snmp.org/docs/man/snmptrapd.html)
- [PRTG SNMP Monitoring Guide](https://www.paessler.com/manuals/prtg/snmp_sensor)

---

## 4. Security Considerations: SNMPv1 vs. v2c vs. v3

SNMP security varies significantly by version. It is critical to understand these differences before deploying SNMP in your environment — misconfigured SNMP can expose sensitive device information or provide an attacker with a read/write interface to network infrastructure.

### Version Comparison

| Feature | SNMPv1 | SNMPv2c | SNMPv3 |
|---|---|---|---|
| **Authentication** | Community string (plaintext) | Community string (plaintext) | Username + MD5 or SHA hash |
| **Encryption** | None | None | AES or DES |
| **Security level** | None | None | noAuthNoPriv / authNoPriv / authPriv |
| **Bulk operations** | No | Yes | Yes |
| **Recommended?** | ❌ No — deprecated | ⚠️ Only in isolated/segmented networks | ✅ Yes — required for any sensitive use |

### SNMPv3 Security Levels Explained

| Level | Auth | Encryption | Use Case |
|---|---|---|---|
| `noAuthNoPriv` | No | No | Development/testing only |
| `authNoPriv` | Yes (MD5/SHA) | No | Low-sensitivity, internal monitoring |
| `authPriv` | Yes (SHA-256) | Yes (AES-256) | Production — recommended for all SMB deployments |

### Hardening Checklist

- [ ] Use **SNMPv3 with `authPriv`** for all production devices
- [ ] Change default community strings (`public`, `private`) immediately on every device
- [ ] Restrict SNMP access to specific management IP addresses using ACLs on network devices
- [ ] Use **read-only** SNMP for monitoring; disable read-write SNMP where not needed
- [ ] Place SNMP traffic on a **dedicated management VLAN** to limit exposure
- [ ] Alert on SNMP **authentication failures** — these can indicate active reconnaissance
- [ ] Regularly audit which devices have SNMP enabled and which versions are in use
- [ ] Block SNMP ports (UDP 161, UDP 162) on perimeter firewalls — SNMP should never be internet-facing

### Key Resources

- [CISA SNMP Security Advisory](https://www.cisa.gov/uscert/ncas/alerts/TA17-156A)
- [NIST Guide to SNMP Security (SP 800-115)](https://csrc.nist.gov/publications/detail/sp/800-115/final)
- [SNMPv3 Configuration Examples (Cisco)](https://www.cisco.com/c/en/us/support/docs/ip/simple-network-management-protocol-snmp/116228-config-snmp-00.html)
- [Net-SNMP SNMPv3 Setup Guide](http://www.net-snmp.org/wiki/index.php/TUT:Simple_SNMPv3_Setup)

---

## Quick Reference: Common SNMP OIDs for SMBs

| OID | Description |
|---|---|
| `1.3.6.1.2.1.1.1.0` | System description (sysDescr) |
| `1.3.6.1.2.1.1.5.0` | Device hostname (sysName) |
| `1.3.6.1.2.1.1.3.0` | System uptime (sysUpTime) |
| `1.3.6.1.2.1.2.2.1.8` | Interface operational status (ifOperStatus) |
| `1.3.6.1.2.1.2.2.1.10` | Interface inbound octets (ifInOctets) |
| `1.3.6.1.2.1.2.2.1.16` | Interface outbound octets (ifOutOctets) |
| `1.3.6.1.2.1.11.30.0` | SNMP authentication failures |
| `1.3.6.1.4.1.318.1.1.1` | APC UPS MIB (battery, load, runtime) |

---

## Additional Resources

| Resource | Link |
|---|---|
| SNMP RFC 3411-3418 (SNMPv3 Standards) | https://datatracker.ietf.org/doc/html/rfc3411 |
| OID Repository / Lookup | https://oid-rep.orange-labs.fr |
| MIB Browser (iReasoning) | https://www.ireasoning.com/mibbrowser.shtml |
| LibreNMS Community | https://community.librenms.org |
| Zabbix Forum | https://www.zabbix.com/forum |

---

*Back to [SIEM Framework Index](../README.md)*
