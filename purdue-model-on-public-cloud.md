---
title: "The Purdue Model on Public Cloud — Not a Compromise, but an Evolution"
date: "2026-06-14"
summary: "Security teams from traditional industries often balk at moving to public cloud. But the Purdue Reference Model was never about physical separation — it's about controlling data flows between functional layers. Public cloud provides richer, more observable tools to implement the same layered security logic."
slug: "purdue-model-on-public-cloud"
tags: ["ICS Security", "OT/IT Convergence", "Cloud Architecture", "IEC 62443", "Purdue Model", "AMI"]
lang: "en"
---

# The Purdue Model on Public Cloud — Not a Compromise, but an Evolution

Security teams from traditional industries tend to grimace the moment someone mentions "moving to public cloud."

The reason is straightforward: we've spent twenty years using the Purdue Reference Model to segment OT networks by layer, with tightly controlled data flows between each level. Now you want me to send SCADA data to AWS, Azure, or Huawei Cloud? Isn't that tearing down my security perimeter?

The concern is valid. The conclusion doesn't have to be.

Public cloud is not the antithesis of the Purdue Model — it offers a more flexible, more observable way to implement the same layered logic. The key is how you use it.

---

## 1. What the Purdue Model Is Actually Protecting

The Purdue Enterprise Reference Architecture (PERA) was introduced by Theodore Williams in the 1990s and later adopted by the ISA-99 / IEC 62443 standards framework, becoming the de facto baseline for industrial control system security.

Its core principle is not "physical isolation" — it's **data flow control between functional layers**:

| Level | Name | Typical Systems |
|-------|------|-----------------|
| Level 0 | Physical Process | Sensors, actuators |
| Level 1 | Control | PLC, DCS, RTU |
| Level 2 | Supervisory | SCADA, HMI |
| Level 3 | Operations | MES, historian |
| DMZ | Demilitarized Zone | Data exchange, firewalls |
| Level 4 | Enterprise | ERP, BI systems |

The essence of the model is: **adjacent layers may communicate, but only through controlled unidirectional or bidirectional interfaces; direct cross-layer access is prohibited.**

Physically separating into different data centers is one way to achieve this goal — but it's not the only way.

---

## 2. How Public Cloud Capabilities Map to Purdue Layers

A public cloud is far more than "renting a server." The networking and access control capabilities it provides are precise enough to map to every layer boundary in the Purdue Model:

**Virtual Private Cloud (VPC) and Subnet Isolation**

Level 3 systems like MES and historian databases, and Level 4 ERP systems, can be deployed in separate VPCs or subnets. Subnets are isolated by default — data flows only when routing rules are explicitly configured. This is exactly the same logic as using VLANs to segment a physical data center.

**Two-Tier Policy: Security Groups and Network ACLs**

Security Groups are stateful firewalls at the instance level; Network ACLs are stateless rules at the subnet level. Combined, they give you precise control over "who can connect to whom, using what protocol, on which port" — which maps directly to the inter-layer access policies in the Purdue Model.

**Cloud-Native DMZ Implementation**

A traditional DMZ is a physically isolated network segment running data diodes or unidirectional gateways. In public cloud, you can build an equivalent DMZ layer using a dedicated subnet + reverse proxy + WAF + traffic mirroring — with even better observability, enabling deep packet inspection and full log retention for every flow crossing the DMZ.

**Private Connectivity (PrivateLink / Dedicated Lines)**

Sending Level 2 SCADA data up to Level 3 in the cloud doesn't require traversing the public internet. Through a cloud provider's dedicated line (e.g., AWS Direct Connect, Azure ExpressRoute, Huawei Cloud dedicated access) or private connectivity services, the link from the plant floor to the cloud can be physically or logically isolated — satisfying the requirement that OT networks not be exposed to the internet.

**IAM and Zero Trust Access Control**

Cloud IAM (Identity and Access Management) enables far more granular control than traditional firewall rules: not just "can this IP connect," but "which service account, at what time, from which source IP, calling which API, operating which piece of data" — full least-privilege principle in practice.

---

## 3. A Typical Reference Architecture

Using manufacturing AMI (Advanced Metering Infrastructure) or smart grid as an example, a typical deployment architecture looks like this:

```
[ Field Layer Level 0-2 ]
  PLC / RTU / Smart Meters
       ↓（Industrial protocols: MQTT/Modbus/IEC 60870）
[ Edge Gateway (on-premises) Level 3 boundary ]
  Data collection, protocol conversion, local buffering
       ↓（Encrypted private link / SD-WAN / PrivateLink）
[ Cloud DMZ Subnet ]
  Message queue, data gateway (no business logic)
       ↓（Internal routing, strict ACL）
[ Cloud Level 3 Subnet ]
  Time-series database (TimescaleDB / InfluxDB)
  Device management platform / alerting engine
       ↓（Application-layer API, read-only interface）
[ Cloud Level 4 Subnet ]
  ERP integration / BI dashboards / external API gateway
```

Each vertical boundary is a controlled enforcement point:
- Edge gateway to cloud travels over a dedicated line, bypassing the public internet
- The DMZ subnet handles only protocol conversion and message relay — it holds no business data
- Between Level 3 and Level 4, only read-only APIs are exposed — no shared databases
- Network access rules for every layer are managed as code (IaC): auditable and rollback-capable

---

## 4. The Real Challenges to Watch Out For

Moving to cloud doesn't automatically mean compliance. A few areas deserve serious attention:

**1. Don't skip edge-side security**

No matter how well you secure the cloud side, if the edge gateway itself has vulnerabilities, attackers can move laterally into Level 2 or even Level 1. The Purdue Model demands more than just "securing the path data takes to the cloud" — it also requires hardening field-side devices and applying network micro-segmentation. Cloud cannot substitute for this work.

**2. Understand the shared responsibility model**

The cloud provider's responsibility boundary stops at the infrastructure layer. Your subnet configurations, security group rules, and IAM policies — those are your responsibility. When auditors come, they're looking at your configuration, not the cloud vendor's compliance certificates.

**3. Centralize logs and observability**

Traditional OT environments often keep logs locally — or have no logging at all. Once you move to cloud, you need to aggregate cloud-side VPC flow logs, API call logs, and security group change logs together with field-side device logs into a unified SIEM platform. Without this, cross-layer attack paths are impossible to trace.

**4. Map compliance standards clause by clause**

IEC 62443, NERC CIP, and GB/T 36323 were written before public cloud was widespread. Some clauses assume physical partitioning. Any cloud migration plan needs a gap analysis that explicitly documents what cloud-native mechanisms are being used as equivalent substitutes for traditional physical isolation. This is the core material for regulatory conversations.

---

## 5. Conclusion: Security Perimeters Were Never Physical

The Purdue Model never specified that "Level 3 must run on a server in your own data center." What it specifies is the **data flow direction and access control policy** between layers.

Public cloud provides stronger tools to implement those policies — finer permission granularity, more complete audit logs, faster policy change response, and compliance evidence that's easier to quantify and verify.

Moving to public cloud isn't a way around the Purdue Model. It's an upgrade — from a network topology diagram to a security policy framework that can be codified, versioned, and automatically validated.

Not a compromise. An evolution.

---

*This article is written for security architects, OT network engineers, and compliance teams evaluating the feasibility of moving industrial control systems to cloud. If you're working on a specific architecture design or an IEC 62443 gap analysis, feel free to leave a comment.*
