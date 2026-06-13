---
title: "Purdue Model on Public Cloud — Not a Compromise, But an Evolution"
date: "2026-06-13"
summary: "Traditional industries moving OT workloads to public cloud fear violating the Purdue Network Security Model. But the Purdue Model defines logical data-flow controls, not physical infrastructure. Public cloud can implement every Purdue boundary more rigorously than on-premises data centers ever could."
slug: "purdue-model-on-public-cloud"
tags: ["OT Security", "Cloud", "ICS", "Purdue Model", "AMI", "Industrial IoT", "IEC 62443"]
lang: "en"
---

# Purdue Model on Public Cloud — Not a Compromise, But an Evolution

Many security teams in traditional industries frown the moment someone mentions moving to public cloud.

The reason is straightforward: we have spent twenty years structuring OT networks around the Purdue Reference Architecture — isolating layers, strictly controlling data flows between them. Now someone wants to send SCADA data to AWS, Azure, or Huawei Cloud? Isn't that dismantling the entire security boundary?

The concern is reasonable. The conclusion is not necessarily correct. Public cloud is not the opposite of the Purdue Model. It provides a more flexible, more observable way to implement the same layered logic. The question is how you use it.

---

## 1. What the Purdue Model Is Actually Protecting

The Purdue Enterprise Reference Architecture (PERA), proposed by Theodore Williams in the 1990s and later adopted by ISA-99 / IEC 62443 as the de facto baseline for industrial control security, has one core principle: **controlled data-flow management between functional levels** — not physical isolation.

| Level | Name | Typical Systems |
|---|---|---|
| Level 0 | Physical Process | Sensors, actuators |
| Level 1 | Control | PLC, DCS, RTU |
| Level 2 | Supervisory | SCADA, HMI |
| Level 3 | Operations | MES, historian |
| DMZ | Demilitarized Zone | Data exchange, firewalls |
| Level 4 | Enterprise | ERP, BI systems |

The essence of the model: adjacent levels may communicate, but only through controlled, unidirectional or explicitly bidirectional interfaces. Direct cross-level access is prohibited. Separating physical data centers is *one way* to achieve this — not the only way.

---

## 2. How Public Cloud Maps to Purdue Zones

A public cloud is far more than renting servers. Its network and access-control capabilities map precisely onto every Purdue boundary.

**VPC and Subnet Isolation**

Level 3 systems (MES, historians) and Level 4 systems (ERP) can be deployed in separate VPCs or subnets. Subnets are isolated by default; data flows only when routing rules are explicitly configured. This is logically identical to VLAN segmentation in a physical data center.

**Two-Layer Policy: Security Groups + Network ACLs**

Security groups provide stateful, instance-level firewall rules. Network ACLs provide stateless, subnet-level rules. Combined, they give precise control over who can reach what, on which protocol, through which port — exactly the inter-level access policies the Purdue Model requires.

**Cloud-Native DMZ**

A traditional DMZ is a physically isolated network segment running data diodes or unidirectional gateways. On public cloud, an equivalent DMZ is built with an isolated subnet, reverse proxy, WAF, and traffic mirroring. The result has *stronger* observability: every packet traversing the DMZ can be deep-packet inspected and logged.

**Private Connectivity (PrivateLink / Dedicated Line)**

Level 2 SCADA data flowing up to cloud-hosted Level 3 does not need to traverse the public internet. Dedicated lines (AWS Direct Connect, Azure ExpressRoute, Huawei Cloud Direct Connect) or private link services provide physically or logically isolated paths from the plant floor to the cloud — satisfying the OT requirement that control networks must never be exposed to the internet.

**IAM and Zero-Trust Access Control**

Cloud IAM delivers finer-grained control than traditional firewall rules: not just whether an IP address can connect, but which service account, at what time, from which source IP, invoking which API, operating which data record. The principle of least privilege can actually be enforced.

---

## 3. A Typical Reference Architecture

Using Advanced Metering Infrastructure (AMI) or smart grid as an example:

```
[ Field Layer — Level 0–2 ]
  PLC / RTU / Smart Meters
       ↓  (IEC 60870 / MQTT / Modbus)
[ Edge Gateway — Level 3 Boundary, on-premises ]
  Data acquisition, protocol conversion, local buffering
       ↓  (Encrypted dedicated line / SD-WAN / PrivateLink)
[ Cloud DMZ Subnet ]
  Message queue, data gateway (no business logic)
       ↓  (Internal routing, strict ACL)
[ Cloud Level 3 Subnet ]
  Time-series database (TimescaleDB / InfluxDB)
  Device management platform / alerting engine
       ↓  (Application-layer API, read-only)
[ Cloud Level 4 Subnet ]
  ERP integration / BI reporting / external API gateway
```

Every boundary in this architecture is controlled:

- Edge-to-cloud traffic travels over a dedicated line, never the public internet
- The DMZ subnet handles protocol translation and message relay only — it holds no business data
- Level 3 to Level 4 exposes read-only APIs, with no shared database access
- All network access rules are managed as Infrastructure-as-Code: auditable, version-controlled, and rollback-capable

---

## 4. The Challenges That Actually Require Attention

Moving to cloud does not automatically mean compliance. Several areas require serious work:

**Edge-side hardening cannot be skipped.** No matter how well the cloud architecture is designed, a vulnerable edge gateway allows lateral movement into Level 2 or even Level 1. The Purdue Model requires securing the field-side devices and applying network micro-segmentation on-premises. Cloud does not replace this work.

**The shared responsibility model must be understood clearly.** The cloud provider's responsibility ends at the infrastructure layer. Subnet configuration, security group rules, and IAM policies are the tenant's responsibility. When auditors review compliance, they inspect your configuration — not the cloud provider's certification documents.

**Logs and observability must be unified.** Traditional OT environments often store logs locally, or generate no logs at all. After moving to cloud, VPC flow logs, API call logs, security group change logs, and field-device logs must all feed into a unified SIEM. Without this, cross-layer attack paths are invisible.

**Compliance standards require line-by-line gap analysis.** IEC 62443, NERC CIP, and GB/T 36323 were written before public cloud became widespread. Some provisions assume physical partitioning. A cloud migration requires a formal gap analysis that explicitly documents which cloud-native mechanism serves as the equivalent substitute for each traditional physical control. This documentation is the core material for conversations with regulators.

---

## 5. What Cloud Adds That Traditional Data Centers Cannot Match

The Purdue Model was designed to be implemented with the tools of its era. Public cloud provides capabilities that make the implementation more rigorous, not less.

**Software-defined policy is auditable by default.** Security group rules and network ACLs are version-controlled, peer-reviewed, and change-logged. No one can bypass a zone boundary by plugging a cable into the wrong switch port.

**Tamper-proof logging at every level.** CloudTrail, Azure Monitor, and equivalent services provide continuous audit logs of every API call, every network connection, every access attempt — across all Purdue levels simultaneously. Most traditional OT networks have nothing comparable.

**Zero-trust maintenance access.** Remote vendor access — historically the Achilles heel of Purdue implementations — can be enforced through identity-verified, session-recorded, time-limited access with no persistent credentials. This is structurally more secure than a shared VPN password to the "OT VLAN."

**Continuous compliance posture verification.** AWS Security Hub, Microsoft Defender for IoT, and Huawei Cloud Security Center continuously verify that zone isolation is maintained, flag configuration drift, and generate audit evidence automatically. Compliance becomes a continuous state, not a point-in-time assessment.

---

## Conclusion: Security Boundaries Were Never Physical

The Purdue Model never required Level 3 to run on a server in your own machine room. It required controlled data-flow direction and access policies between levels.

Public cloud provides stronger tools to implement those policies: finer permission granularity, more complete audit logs, faster policy-change response, and compliance evidence that is easier to quantify and verify.

Moving OT workloads to public cloud is not a way around the Purdue Model. It is an upgrade — from a network topology diagram into a security policy framework that can be codified, versioned, and automatically verified.

Not a compromise. An evolution.

---

*This article is intended for security architects, OT network engineers, and compliance teams evaluating the feasibility of moving industrial control systems to public cloud. If you are working on a specific architecture design or IEC 62443 gap analysis, feel free to leave a comment.*
