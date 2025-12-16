# CNIT 441 – Enterprise Network Design, Automation & AWS Project

## Project Overview

This project demonstrates the design, implementation, and automation of a **dual-stack enterprise network** connecting multiple campus buildings to an ISP using industry-standard routing, redundancy, and management practices. The solution focuses on **scalability, resiliency, and operational verification**, with automation used to configure and validate core network services.

The network was designed to meet real-world enterprise requirements, including internal routing, internet connectivity, first-hop redundancy, and centralized management readiness.

---

## Network Architecture Summary

The topology consists of:

### Access Layer
- **A1 switch**
- Provides end-device connectivity
- VLAN-based segmentation for different departments

### Distribution Layer
- **D1 and D2 multilayer switches**
- Inter-VLAN routing
- **HSRPv2** for default gateway redundancy
- **EtherChannel (LACP)** for link redundancy

### Routing Layer
- **R1** – Main HQ router
- **R3** – Secondary building router
- **R2** – ISP router (simulated internet edge)

### ISP Connectivity
- **BGP (MP-BGP)** used between enterprise routers and ISP
- Enterprise ASN advertises internal IPv4 networks
- ISP advertises a default route to the enterprise

---

## Routing & Protocols

### Internal Routing
- **OSPFv2** for IPv4 internal routing
- **OSPFv3** for IPv6 internal routing
- Full internal reachability across all VLANs and buildings

### External Routing
- **BGP (MP-BGP)** used for ISP connectivity
- IPv4 BGP advertises enterprise networks to the ISP
- ISP provides a default route to enterprise routers
- IPv6 BGP sessions are configured for future readiness
- IPv6 internet routing is not fully enabled due to project scope and time constraints

---

## High Availability & Redundancy

- **HSRPv2** on D1 and D2
  - Active/Standby gateways per VLAN
  - Load balancing across VLANs
- **IP SLA + Object Tracking**
  - Monitors upstream reachability
  - Triggers HSRP failover during connectivity loss
- **EtherChannel (LACP)**
  - Redundant Layer 2 switch links
- **Rapid PVST+**
  - Optimized spanning-tree root placement

---

## Addressing & Services

- **IPv4 & IPv6 Dual Stack**
  - IPv4 used for primary internet connectivity
  - IPv6 fully enabled for internal routing
- **DHCP**
  - Distributed DHCP services on D1 and D2
  - Redundant address allocation per VLAN
- **Management VLAN**
  - Dedicated VLAN for device management and automation access

---

## Automation Strategy (Ansible)

Automation is used to **configure, verify, and validate** the network in a controlled and repeatable manner.

### Configuration Playbooks
- HSRP configuration (IPv4 & IPv6)
- NTP configuration
- Banner configuration
- Basic network service setup

### Verification Playbooks
- Routing verification (OSPF & BGP)
- Switch operational checks (VLANs, trunks, EtherChannels)
- Connectivity and ping testing
- DHCP lease verification
- NTP, logging, SNMP, and AAA verification

> **Note:** AAA configuration was intentionally not fully automated to avoid administrative lockout risks. AAA was configured manually with local fallback authentication and verified using automation. This reflects real-world enterprise change-management best practices.

---

## Monitoring & Management Readiness

- NTP synchronization across all devices
- Syslog and SNMP readiness for centralized monitoring
- Design supports future integration with AWS-hosted logging and monitoring servers

---

## Project Goals Met

- Layer 2 redundancy (EtherChannel, RSTP)
- IPv4 and IPv6 dual-stack implementation
- OSPF internal routing (OSPFv2 & OSPFv3)
- BGP ISP connectivity
- Gateway redundancy with HSRP
- Automated configuration and verification
- Scalable and enterprise-ready design

---

## Design Notes

- IPv4 was prioritized for internet connectivity to simplify failover demonstrations and verification.
- IPv6 is fully supported internally and positioned for future internet expansion.
- Automation emphasizes **verification and consistency** rather than unsafe bulk configuration.

---

## Conclusion

This project demonstrates a realistic enterprise network deployment combining **robust routing design, redundancy, and automation**. The solution balances technical depth with operational safety, reflecting best practices used in real production environments.
