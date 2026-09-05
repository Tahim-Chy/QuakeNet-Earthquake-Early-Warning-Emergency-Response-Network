# QuakeNet — Earthquake Early Warning & Emergency Response Network

![Course](https://img.shields.io/badge/Course-CSE421-blue) ![Platform](https://img.shields.io/badge/Platform-Cisco%20Packet%20Tracer-1BA0D7) ![Status](https://img.shields.io/badge/Status-Complete-brightgreen)

A 7-site wide-area network, designed and built in Cisco Packet Tracer, connecting the functional units of a national earthquake early-warning and emergency-response system — VLSM addressing, RIPv2 with redistribution, floating and recursive static backup routes, three different DHCP delivery methods, and per-unit email/web/DNS services.

**CSE421 — Computer Network | Section 07 | Group 703**

---

## Table of Contents

- [Overview](#overview)
- [Topology](#topology)
- [Key Design Features](#key-design-features)
- [IP Addressing (VLSM)](#ip-addressing-vlsm)
- [Routing Design](#routing-design)
- [DHCP](#dhcp)
- [Services](#services)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Documentation](#documentation)
- [Team](#team)

---

## Overview

QuakeNet connects seven functional units into one resilient network:

| Unit | Role |
|---|---|
| **NEOC** | National Emergency Operation Center |
| **SMS** | Seismic Monitoring Station |
| **FRH** | Fire & Rescue Headquarters |
| **MEC** | Medical Emergency Center |
| **USR** | Urban Search & Rescue Camp |
| **CRC** | Communication Relay Center |
| **MOP** | Mountain Observation Post |

Each unit is represented by one router with its own LAN (switch, 2 PCs, printer, server) — **43 devices** in total, tied together with a mix of shared-switch, direct point-to-point, and stub links designed around one guiding idea: the most time-critical alerting path (NEOC ↔ SMS ↔ FRH) gets deliberate redundancy that the rest of the network doesn't.

## Topology

![QuakeNet Topology](assets/topology.png)

- **Warning loop** — NEOC, SMS, and FRH are connected in a redundant triangle: three direct point-to-point links, layered *on top of* the shared switch connections NEOC and FRH already have. If any one link fails, RIP reconverges and the alert path keeps working.
- **Central switch backbone** — NEOC, FRH, MEC, and CRC share one central switch.
- **Direct link** — NEOC and MEC also have a dedicated point-to-point connection, independent of the central switch.
- **Stub links** — USR connects only to MEC; MOP connects only to CRC.

## Key Design Features

- **ID-derived addressing** — the base network (`11.35.0.0/16`) and a custom Administrative Distance (`144`) are both mathematically derived from group members' student IDs, not arbitrary.
- **Full VLSM subnetting** — 14 subnets carved from a single `/16`, each sized to its exact host requirement, from a `/19` (5000 hosts) down to `/30` point-to-point links.
- **Hybrid routing** — RIPv2 with static-route redistribution on the warning-loop routers, static-only routing everywhere else, including floating *and* recursive backup routes at a custom administrative distance.
- **Three DHCP delivery methods in one design** — router-based (native IOS) on NEOC, server-based on MEC/CRC, and DHCP relay (`ip helper-address`) on SMS/FRH/USR/MOP.
- **Full service stack** — email on all seven sites (with verified cross-domain delivery), web on two, and centralized DNS resolving every hostname network-wide.
- **A real routing nuance, documented not hidden** — CRC's floating backup route shares a Layer-2 segment with its primary path, which means it doesn't fail over automatically the way a textbook floating static route would. Rather than "fixing" this by changing the topology, the report documents *why* it behaves this way and how it was verified.

## IP Addressing (VLSM)

Base network: **11.35.0.0/16**

| Segment | Hosts req. | Prefix | Network | Usable range |
|---|---|---|---|---|
| NEOC-LAN | 5000 | /19 | 11.35.0.0 | 11.35.0.1 – 11.35.31.254 |
| USR-LAN | 1040 | /21 | 11.35.32.0 | 11.35.32.1 – 11.35.39.254 |
| MEC-LAN | 500 | /23 | 11.35.40.0 | 11.35.40.1 – 11.35.41.254 |
| FRH-LAN | 320 | /23 | 11.35.42.0 | 11.35.42.1 – 11.35.43.254 |
| SMS-LAN | 180 | /24 | 11.35.44.0 | 11.35.44.1 – 11.35.44.254 |
| CRC-LAN | 100 | /25 | 11.35.45.0 | 11.35.45.1 – 11.35.45.126 |
| MOP-LAN | 50 | /26 | 11.35.45.128 | 11.35.45.129 – 11.35.45.190 |
| Central switch | 4 | /29 | 11.35.45.192 | 11.35.45.193 – 11.35.45.198 |
| 6× P2P links | 2 each | /30 | 11.35.45.200 – .223 | see report for full breakdown |

The full branching VLSM tree — every split from the /16 down to each allocated subnet — is in [`assets/vlsm-tree.png`](assets/vlsm-tree.png).

## Routing Design

- **RIPv2 domain:** NEOC, SMS, and FRH run RIPv2 over the warning-loop links, with LAN-facing and backbone interfaces set `passive-interface`.
- **Redistribution:** NEOC holds static routes to every network outside the RIP domain and redistributes them into RIP — the only way SMS and FRH learn the rest of the network exists.
- **Floating & recursive backups:** MEC and CRC each carry a primary static route plus a floating backup at administrative distance `144`, calculated from a group member's ID rather than picked arbitrarily.
- **Static-only stubs:** USR and MOP use exit-interface static routes (a deliberate contrast with MEC/CRC's recursive, next-hop-based routes). No default routes anywhere in the design.

Example — one of the floating backup routes, only ever installed if the primary path fails:
```
ip route 11.35.0.0 255.255.224.0 11.35.45.196 144
```

## DHCP

| Method | Where | Serves |
|---|---|---|
| Router-based (IOS) | NEOC | NEOC-LAN, SMS-LAN, FRH-LAN |
| Server-based | MEC-SRV | MEC-LAN, USR-LAN |
| Server-based | CRC-SRV | CRC-LAN, MOP-LAN |
| Relay (`ip helper-address`) | SMS, FRH, USR, MOP | forwards to the above |

## Services

- **Email** — every site runs its own mail domain (`mail.<unit>.gov`), two mailboxes each; NEOC ↔ MEC cross-domain delivery verified.
- **Web** — NEOC and MEC serve status pages at `www.neoc.alert.gov` and `www.mec.alert.gov`.
- **DNS** — NEOC hosts the network's only DNS server, authoritative for every mail and web hostname above.

## Repository Structure

```
QuakeNet/
├── README.md
├── QuakeNet.pkt                        # Cisco Packet Tracer network file
├── docs/
│   ├── QuakeNet_Report.pdf             # Full project report
│   ├── QuakeNet_Report.docx
│   └── QuakeNet_Work_Distribution.pdf
└── assets/
    ├── topology.png                    # Packet Tracer topology screenshot
    └── vlsm-tree.png                   # Full VLSM branching diagram
```

## Getting Started

1. Install [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) (free with a Cisco Networking Academy account).
2. Clone this repo and open `QuakeNet.pkt`.
3. See [`docs/QuakeNet_Report.pdf`](docs/QuakeNet_Report.pdf) for the full design rationale, every router's complete configuration, and end-to-end verification results.

## Documentation

- 📄 [Full Project Report](docs/QuakeNet_Report.pdf) — topology, VLSM tree, complete router configurations, DHCP/routing/service design, and test verification
- 📄 [Work Distribution](docs/QuakeNet_Work_Distribution.pdf)

## Team

| Name | Student ID | Role |
|---|---|---|
| Tahamidul Alam Chowdhury | 22299066 | Design & Documentation Lead |
| Bornil Barua | 22201838 | Build & Infrastructure |
| Tabia Sultana Ritu | 23201510 | Routing & DHCP |
| Syed Kafi | 23101135 | Services & Verification |

*Project submitted for CSE421 (Computer Network), Section 07, Group 703.*
