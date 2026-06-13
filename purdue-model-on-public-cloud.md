---
title: "Purdue Model on Public Cloud — Not a Compromise, But an Evolution"
date: "2026-06-13"
summary: "Traditional industries are moving OT workloads to public cloud while still required to follow the Purdue Network Security Model. This article argues that the Purdue Model is a logical security framework, not a physical infrastructure prescription — and public cloud can implement it more rigorously than on-premises data centers ever could."
slug: "purdue-model-on-public-cloud"
tags: ["OT Security", "Cloud", "ICS", "Purdue Model", "AMI", "Industrial IoT"]
lang: "en"
---

# Purdue Model on Public Cloud — Not a Compromise, But an Evolution

For decades, the Purdue Enterprise Reference Architecture (PERA) — commonly known as the Purdue Model — has been the de facto blueprint for securing industrial control systems (ICS) and operational technology (OT) networks. Its layered hierarchy, from physical processes at Level 0 up to enterprise IT at Level 4/5, defined how utilities, manufacturers, and critical infrastructure operators thought about network segmentation.

Then public cloud arrived. And a generation of security engineers said: "The Purdue Model and the cloud are incompatible. You can't put OT in someone else's data center."

That instinct is understandable. It is also wrong — or at least, it is answering the wrong question.

---

## What the Purdue Model Actually Says

The Purdue Model is a *logical* security framework, not a *physical* infrastructure prescription.

Its core principle is **zone isolation with controlled data flows**: each level communicates only with adjacent levels, unidirectional data flows are enforced where possible, and no direct path exists between the enterprise IT network and the plant floor. The model defines *what talks to what* and *how*, not *where the servers physically sit*.

When engineers insist that Purdue compliance requires separate physical data centers — one for IT, one for OT — they are conflating the logical model with a specific historical implementation of it. In the 1990s and 2000s, physical separation was the most practical way to enforce zone isolation. Air gaps and firewalls between buildings were the tools available.

Public cloud gives you different tools — in many ways, better ones.

---

## The Traditional Implementation's Hidden Weaknesses

Before arguing that cloud can implement Purdue, it is worth being honest about how poorly many on-premises implementations actually achieve it.

**Physical separation does not equal logical isolation.** Separate server rooms connected by a misconfigured firewall rule are not meaningfully isolated. A VPN tunnel from a vendor's laptop to the "OT network" punches through every physical boundary you built. The Colonial Pipeline attack in 2021 compromised an IT billing system and reached OT through exactly this kind of implicit trust across supposedly separate networks.

**Maintenance access creates permanent holes.** Remote access for vendors and engineers is a necessity in any industrial operation. In traditional implementations, this access is often managed with shared credentials, persistent VPN connections, and minimal logging — because the physical security model never anticipated it properly.

**Visibility is fragmentary.** Traditional OT networks are notoriously opaque. Traffic between Levels 1 and 2 often traverses unmanaged switches with no logging. Security teams cannot see what they cannot monitor.

Physical separation gave the *appearance* of Purdue compliance while leaving fundamental logical isolation problems unaddressed.

---

## How Public Cloud Implements Purdue — Layer by Layer

Let's walk through each Purdue level and show how public cloud infrastructure maps onto it.

### Level 0–1: Field Devices and Controllers (Stay On-Premises)

Nothing in the Purdue Model requires Level 0 (sensors, actuators) or Level 1 (PLCs, RTUs) to move to the cloud — and they should not. These devices are physically embedded in the plant or grid infrastructure. The Purdue boundary here is the same as always: Level 1 to Level 2 communication is tightly controlled.

What changes is that the **demarcation point** — the data diode, protocol gateway, or historian — can now push data upward into a cloud-hosted Level 2/3 zone rather than an on-premises one.

### Level 2: Supervisory Control (Cloud-Hosted, Air-Gapped VPC)

In a public cloud implementation, Level 2 lives in a **dedicated, isolated Virtual Private Cloud (VPC)** with:

- No inbound internet routes
- Outbound-only data flows to Level 3 (enforced by security group rules, not firewall appliances)
- All traffic logged to an immutable audit store
- No peering with the enterprise IT VPC

This is logically equivalent to a physically isolated Level 2 network — and in practice, it is *more* reliably isolated, because cloud security groups are software-defined policy that cannot be bypassed by plugging a cable into the wrong switch.

### Level 3: Operations / Historian (Cloud-Hosted, Controlled DMZ)

Level 3 — the operations network, data historians, manufacturing execution systems — maps to a **separate VPC** with explicit, whitelisted peering rules to Level 2 (inbound data only) and Level 4 (outbound reports only).

This is precisely the DMZ logic the Purdue Model specifies. The cloud implementation enforces it through:

- VPC peering with explicit allow-lists
- AWS PrivateLink / Azure Private Endpoint (no data traverses public internet even between cloud zones)
- IAM role-based access: no human has standing access to Level 3; access is granted via just-in-time (JIT) elevation with full audit trail

### Level 4/5: Enterprise IT and Business Network

Standard enterprise cloud: ERP, analytics, dashboards. Connected to Level 3 via one-way data replication (Kafka, event streaming) — Level 4 can read historian data; it cannot send commands downward.

This unidirectional data flow — the most critical Purdue principle — is *easier* to enforce in cloud infrastructure than in traditional networks, because it is implemented as an event stream architecture, not a firewall rule that someone can accidentally modify.

---

## What Cloud Adds That Traditional Data Centers Cannot

The Purdue Model was designed to be implemented with the tools of its era. Public cloud provides capabilities that make implementation more rigorous, not less:

**Software-defined network policy.** Security group rules and network ACLs are version-controlled, peer-reviewed, and auditable. No one can plug a cable into the wrong VLAN. Every change is logged, attributable, and reversible.

**Immutable logging.** CloudTrail, Azure Monitor, and equivalent services provide tamper-proof audit logs of every API call, every network connection, every access attempt — across all Purdue levels simultaneously. Traditional OT networks rarely have equivalent visibility.

**Zero-trust access to maintenance pathways.** Remote vendor access — the Achilles heel of traditional Purdue implementations — can be enforced through cloud-native zero-trust frameworks: identity-verified, session-recorded, time-limited, with no persistent credentials. This is structurally more secure than a shared VPN password to the "OT VLAN."

**Automated compliance posture.** AWS Security Hub, Azure Defender for IoT, and equivalent tools continuously verify that zone isolation is maintained, flag drift, and generate audit evidence. Compliance is not a point-in-time assessment; it is a continuous state.

**Physical redundancy without physical complexity.** Multi-AZ deployments provide the hardware redundancy that critical OT systems require, without building and maintaining multiple data centers.

---

## The Real Question Is Not "Cloud or Not?" — It's "Which Threat Model?"

The argument against cloud-hosted OT environments is usually framed as: "What if the cloud provider is breached?"

This is a legitimate question. It is also worth asking the parallel question: "What if the on-premises data center is breached?" — because that happens too, more often and with less detection capability.

The Purdue Model was never about making a system impenetrable. It was about **containing blast radius**: if one zone is compromised, the attacker cannot move laterally to adjacent zones. That principle applies identically on public cloud. An attacker who compromises the enterprise IT VPC (Level 4/5) cannot reach the OT VPC (Level 2/3) if the zone isolation is correctly implemented — regardless of whether both VPCs run on the same cloud provider's hardware.

The relevant question is not the physical location of the infrastructure. It is whether the logical boundaries are correctly defined, correctly implemented, and continuously verified. Public cloud infrastructure makes all three of those tasks more tractable, not less.

---

## A Note for Regulated Industries

Industries operating under IEC 62443, NERC CIP, or equivalent standards sometimes interpret compliance as requiring physical separation of OT and IT environments. Regulators and auditors are increasingly recognizing that logical separation — properly documented, continuously monitored, and independently verified — satisfies the intent of these frameworks.

The key is documentation: a clear mapping of Purdue levels to cloud infrastructure components, explicit data flow diagrams, and audit evidence from the cloud platform's logging and monitoring systems. Compliance is an argument you make to an auditor, and cloud infrastructure provides more evidence to make that argument with than most traditional OT environments ever could.

---

## Conclusion

The Purdue Model is not a blueprint for building separate data centers. It is a framework for defining security zones and controlling data flows between them.

Public cloud does not violate that framework. Implemented correctly, it enforces it more rigorously — with software-defined policy that cannot be bypassed by a mis-patched cable, with immutable logging that provides continuous audit evidence, and with zero-trust access controls that address the maintenance pathway vulnerabilities that have always been the practical weak point of traditional Purdue implementations.

The question for industrial operators moving to public cloud is not "Can we maintain Purdue compliance?" The answer to that is yes. The question is "Are we implementing the logical isolation correctly?" — and that question applies equally to on-premises infrastructure.

The Purdue Model belongs in the cloud. That is not a compromise. It is where the model was always heading.
