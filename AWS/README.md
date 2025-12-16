# Cisco Lab Logging + Dashboard on AWS (Syslog + SNMP Traps + Grafana)

This project stands up a lightweight, student-friendly monitoring setup in AWS for a Cisco lab network using **push-based telemetry**. Cisco devices send **syslog** and **SNMP traps** outbound to an AWS EC2 instance. The EC2 instance hosts a web UI (Grafana) and log storage/search (Loki), plus an uptime-percentage view for interfaces derived from link up/down events.

This design works well behind your Home Internet / CGNAT because AWS does **not** need to initiate inbound connections to the home network; the Cisco devices push events to AWS.

---

## Architecture

**Data flow (high level):**

- Cisco routers/switches → **Syslog (UDP 514)** → EC2 → `rsyslog` → `/var/log/cisco-lab.log`
- Cisco routers/switches → **SNMP traps (UDP 162)** → EC2 → `snmptrapd` → syslog → `/var/log/snmptraps.log`
- EC2 → **Promtail** tails log files and ships to **Loki**
- **Grafana** queries Loki for logs and displays dashboards
- **Promtail** also derives a metric `cisco_interface_up{device,iface}` from syslog linkUp/linkDown lines
- **Prometheus** scrapes Promtail metrics; Grafana uses Prometheus to compute **interface uptime %**

**Components:**
- `rsyslog`: receives syslog over UDP/514 and writes to a log file
- `snmptrapd`: receives SNMP traps over UDP/162 and logs to syslog
- `promtail`: log shipper (tails files, adds labels, pushes to Loki); also emits interface state metrics
- `loki`: log database/index for Grafana
- `prometheus`: time-series DB for interface uptime metrics
- `grafana`: web UI for logs and dashboards

---

## What you get

- Centralized syslog collection in AWS
- Centralized SNMP trap collection in AWS
- Grafana log explorer and dashboards
- Per-device/per-interface uptime percentage panels (derived from link up/down events)
- Works for multiple devices even when interface names overlap (e.g., multiple `Gi0/1`) via `device` label extraction

---

## Prerequisites

- AWS account with EC2 access
- A Cisco lab device (IOS/IOS-XE) with Internet access
- Ability to configure Cisco `logging host` and `snmp-server host`
- Your home public IP (recommended for restricting inbound SG rules)

---

## Quick Start (high level)

1. **Step 1:** Launch an EC2 instance and open inbound ports:
   - TCP 22 (SSH), UDP 514 (syslog), UDP 162 (SNMP traps), TCP 3000 (Grafana)

2. **Steps 2 & 3:** Configure EC2 host services:
   - `rsyslog` listens on UDP/514 and writes `/var/log/cisco-lab.log`
   - `snmptrapd` listens on UDP/162; `rsyslog` routes traps to `/var/log/snmptraps.log`

3. **Steps 4 & 5:** Deploy the Docker stack:
   - Loki + Promtail + Grafana + Prometheus (docker compose)

4. **Grafana setup:**
   - Add Loki data source (`http://loki:3100`)
   - Add Prometheus data source (`http://prometheus:9090`)
   - Import/create dashboards for syslog, traps, and uptime %

5. **Cisco config:**
   - Set unique `hostname` on each device and enable `logging origin-id hostname`
   - Send syslog to EC2 public IP
   - Send SNMP traps to EC2 public IP
   - Generate a test event (interface flap) to verify ingestion

---

## Documentation Files in /Instructions

- `aws-cisco-logging-step-1.md`  
  Launch EC2 + Security Group rules + SSH test

- `aws-cisco-logging-steps-2-3.md`  
  EC2 host configuration: rsyslog + snmptrapd (syslog + SNMP traps)

- `aws-cisco-logging-steps-4-5.md`  
  Docker stack deployment: Loki/Promtail/Grafana/Prometheus + metric extraction

- `aws-cisco-logging-grafana-setup.md`  
  Grafana data sources + log dashboards + uptime % panels (PromQL)

---

## Key Ports

Inbound to EC2 (Security Group):
- `22/tcp` SSH (restrict to your IP)
- `3000/tcp` Grafana UI (restrict to your IP)
- `514/udp` Cisco syslog (restrict to your IP)
- `162/udp` SNMP traps (restrict to your IP)

Internal (Docker network):
- Loki: `3100/tcp`
- Promtail metrics: `9080/tcp`
- Prometheus: `9090/tcp`

---

## Notes / Limitations

- This is **event-driven monitoring** (syslog + traps). It does not perform full SNMP polling from AWS unless you add a VPN/tunnel.
- Interface uptime is derived from link up/down messages. If logs do not arrive, uptime metrics may not reflect reality.
- EC2 public IP can change if the instance is stopped/started. Update Cisco configs accordingly, or use a stable addressing method if required.

---

## Cleanup

When done:
- Stop/terminate the EC2 instance
- Remove or tighten any overly-permissive Security Group rules
- Delete Docker volumes if you want to fully remove stored logs/metrics

---
