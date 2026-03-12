# Multi-Site WAN — Cisco Physical Gear Lab

## Table of Contents
1. [Overview](#overview)
2. [Hardware & Software](#hardware--software)
3. [Network Topology](#network-topology)
4. [IP Addressing](#ip-addressing)
5. [Device Roles & Configuration Summary](#device-roles--configuration-summary)
   - [Core Routers (P Routers)](#core-routers-p-routers)
   - [PE Routers](#pe-routers)
   - [Aggregation Switch](#aggregation-switch)
6. [Routing Protocols](#routing-protocols)
7. [MPLS & LDP](#mpls--ldp)
8. [VRF & L3VPN Design](#vrf--l3vpn-design)
9. [BGP Design](#bgp-design)
10. [Traffic Flow](#traffic-flow)
11. [Inter-Client Connectivity Matrix](#inter-client-connectivity-matrix)

---

## Overview

This lab simulates a **service-provider / enterprise WAN** built on physical Cisco gear. It implements a full **MPLS L3VPN** backbone with three sites, each served by a dedicated PE (Provider Edge) router. A triangular core (P-router mesh) provides redundant label-switched paths across the backbone. Five customer VRFs (`CLIENT1`–`CLIENT5`) represent isolated branch sites, and a sixth VRF (`DC_CENTRAL`) acts as a shared Data-Centre hub that all clients can reach.

---

## Hardware & Software

| Device | Model | OS |
|--------|-------|----|
| R1C, R2C, R3C | Cisco ISR 4331 | IOS-XE 16.x (Fuji) – 17.x (Bengaluru) |
| R1E, R2E, R3E | Cisco ISR 4331 | IOS-XE 16.x (Fuji) – 17.x (Bengaluru) |
| SW_AGR1 | Cisco Catalyst 3750-E | IOS (IP Services) |

> **Note:** SW_AGR2 and SW_AGR3 are referenced in the PE configs but their configuration files are not yet included in this repository. SW_AGR1 is the only switch with a config file present.

---

## Network Topology

```
                         ┌─────────────────────────────────────────────────┐
                         │               MPLS CORE (AS 65000)              │
                         │                                                  │
                         │          10.0.12.x /30                          │
                         │    R1C ──────────────────── R2C                 │
                         │   (P)  10.255.1.1     10.255.2.2  (P)           │
                         │    │  \                        /  │              │
                         │    │   \ 10.0.13.x /30        /   │              │
                         │    │    \      10.0.23.x /30 /    │              │
                         │    │     \                  /     │              │
                         │    │      └──── R3C ────────      │              │
                         │    │           (P)                │              │
                         │    │        10.255.3.3            │              │
                         │    │                              │              │
                         │ 10.0.101.x /30            10.0.102.x /30        │
                         │    │                              │              │
                         │   R1E                            R2E             │
                         │   (PE)                          (PE)             │
                         │ 10.255.11.11               10.255.12.12          │
                         │    │                              │              │
                         │    │                       10.0.103.x /30        │
                         │    │                              │              │
                         │    │                            R3E              │
                         │    │                           (PE)              │
                         │    │                       10.255.13.13          │
                         └────┼──────────────────────────────┼─────────────┘
                              │                              │
                        ┌─────┴──────┐               ┌──────┴─────┐
                        │  SW_AGR1   │               │  SW_AGR2   │
                        │  (3750-E)  │               │  (3750-E)  │
                        └─────┬──────┘               └──────┬─────┘
                 VLAN 101 ─┬──┘                             └──┬─ VLAN 103
                 VLAN 102 ─┘                                   └─ VLAN 104
                 CLIENT1 Site                           CLIENT3 / CLIENT4 Sites
                 CLIENT2 Site

                                              SW_AGR3 (3750-E) — attached to R3E
                                                 VLAN 105 → CLIENT5 Site
                                                 VLAN 200 → DC_CENTRAL Hub
```

### Physical link summary

| Link | Device A | Interface | IP A | Device B | Interface | IP B |
|------|----------|-----------|------|----------|-----------|------|
| Core | R1C | Gi0/0/0 | 10.0.12.1/30 | R2C | Gi0/0/0 | 10.0.12.2/30 |
| Core | R1C | Gi0/0/1 | 10.0.13.1/30 | R3C | Gi0/0/0 | 10.0.13.2/30 |
| Core | R2C | Gi0/0/1 | 10.0.23.1/30 | R3C | Gi0/0/1 | 10.0.23.2/30 |
| P-PE | R1C | Gi0/0/2 | 10.0.101.1/30 | R1E | Gi0/0/2 | 10.0.101.2/30 |
| P-PE | R2C | Gi0/0/2 | 10.0.102.1/30 | R2E | Gi0/0/2 | 10.0.102.2/30 |
| P-PE | R3C | Gi0/0/2 | 10.0.103.1/30 | R3E | Gi0/0/2 | 10.0.103.2/30 |
| PE-AGR | R1E | Gi0/0/0 (trunk) | — | SW_AGR1 | Gi1/0/1 | 172.16.1.1/30 |
| PE-AGR | R2E | Gi0/0/0 (trunk) | — | SW_AGR2 | — | — |
| PE-AGR | R3E | Gi0/0/0 (trunk) | — | SW_AGR3 | — | — |

---

## IP Addressing

### Loopbacks (Router-IDs & LDP IDs)

| Device | Loopback0 |
|--------|-----------|
| R1C | 10.255.1.1/32 |
| R2C | 10.255.2.2/32 |
| R3C | 10.255.3.3/32 |
| R1E | 10.255.11.11/32 |
| R2E | 10.255.12.12/32 |
| R3E | 10.255.13.13/32 |

### Core P-to-P Links (MPLS-enabled)

| Subnet | Endpoints |
|--------|-----------|
| 10.0.12.0/30 | R1C ↔ R2C |
| 10.0.13.0/30 | R1C ↔ R3C |
| 10.0.23.0/30 | R2C ↔ R3C |
| 10.0.101.0/30 | R1C ↔ R1E |
| 10.0.102.0/30 | R2C ↔ R2E |
| 10.0.103.0/30 | R3C ↔ R3E |

### PE-to-CE / PE-to-AGR Links (VRF sub-interfaces)

| VRF | VLAN | PE Sub-if IP | CE/AGR IP | PE Router |
|-----|------|--------------|-----------|-----------|
| CLIENT1 | 101 | 172.16.1.1/30 | 172.16.1.2/30 | R1E |
| CLIENT2 | 102 | 172.16.2.1/30 | 172.16.2.2/30 | R1E |
| CLIENT3 | 103 | 172.16.3.1/30 | 172.16.3.2/30 | R2E |
| CLIENT4 | 104 | 172.16.4.1/30 | 172.16.4.2/30 | R2E |
| CLIENT5 | 105 | 172.16.5.1/30 | 172.16.5.2/30 | R3E |
| DC_CENTRAL | 200 | 172.16.100.1/30 | 172.16.100.2/30 | R3E |

### SW_AGR1 — CE-side (VRF-Lite)

| VRF | Interface | IP |
|-----|-----------|-----|
| CLIENT1 | Gi1/0/10 | 172.16.11.2/30 (toward client LAN) |
| CLIENT2 | Gi1/0/11 | 172.16.12.2/30 (toward client LAN) |
| Uplink to R1E | Gi1/0/1 | 172.16.1.1/30 |

---

## Device Roles & Configuration Summary

### Core Routers (P Routers)

`R1C`, `R2C`, and `R3C` form the **MPLS label-switching core**. They do **not** hold customer VRF state — they only forward labeled packets. All three are fully meshed via /30 point-to-point links with **MPLS/LDP** enabled on every core-facing interface including the P-PE links.

| Feature | Value |
|---------|-------|
| Routing protocol | OSPF process 1, Area 0 |
| OSPF networks | All 10.0.0.0/8 + Loopback0 |
| MPLS protocol | LDP |
| LDP Router-ID | Loopback0 (forced) |
| VRF awareness | None (pure P role) |

### PE Routers

`R1E`, `R2E`, and `R3E` are the **Provider Edge** routers implementing L3VPN. Each PE:
- Participates in OSPF Area 0 (backbone transport only — Loopback and P-PE link)
- Runs **LDP** toward its directly connected P router
- Maintains per-client **VRF instances** with Route Distinguishers and Route Targets
- Peers with the other two PEs via **iBGP (AS 65000)** in the `vpnv4` address-family using Loopback0 as update-source
- Connects to the aggregation switch via **802.1Q sub-interfaces** (one VLAN per VRF)
- Runs **eBGP** within each VRF to exchange client prefixes with the CE/aggregation device

#### R1E — Site 1

| VRF | RD | Export RT | Import RTs | CE peer (eBGP) | CE AS |
|-----|----|-----------|------------|----------------|-------|
| CLIENT1 | 65000:1 | 65000:1 | 65000:1, 65000:100 | 172.16.1.2 | 65021 |
| CLIENT2 | 65000:2 | 65000:2 | 65000:2, 65000:100 | 172.16.2.2 | 65021 |

#### R2E — Site 2

| VRF | RD | Export RT | Import RTs | CE peer (eBGP) | CE AS |
|-----|----|-----------|------------|----------------|-------|
| CLIENT3 | 65000:3 | 65000:3 | 65000:3, 65000:100 | 172.16.3.2 | 65022 |
| CLIENT4 | 65000:4 | 65000:4 | 65000:4, 65000:100 | 172.16.4.2 | 65022 |

#### R3E — Site 3 + DC

| VRF | RD | Export RT | Import RTs | CE peer (eBGP) | CE AS |
|-----|----|-----------|------------|----------------|-------|
| CLIENT5 | 65000:5 | 65000:5 | 65000:5, 65000:100 | 172.16.5.2 | 65023 |
| DC_CENTRAL | 65000:100 | 65000:100 | 65000:100, 65000:1–5 | 172.16.100.2 | 65023 |

### Aggregation Switch

`SW_AGR1` (Catalyst 3750-E) implements **VRF-Lite** on the CE side. It:
- Maintains one VRF per client (`CLIENT1`, `CLIENT2`) using IOS-style `ip vrf` (pre-named VRF syntax, compatible with 3750-E IOS)
- Runs **eBGP (AS 65021)** per-VRF toward R1E (AS 65000)
- Provides per-client LAN-facing interfaces (`Gi1/0/10` for CLIENT1, `Gi1/0/11` for CLIENT2)
- The uplink to R1E (`Gi1/0/1`) carries traffic that is VLAN-tagged on the PE side (R1E uses sub-interfaces), while SW_AGR1 itself uses a plain routed port — the 802.1Q encapsulation terminates on the PE sub-interfaces

---

## Routing Protocols

### OSPF — Backbone IGP

All 6 routers participate in **OSPF process 1, Area 0** (single area). OSPF is used exclusively as the transport IGP to distribute Loopback reachability, enabling:
- LDP sessions between all MPLS nodes
- iBGP sessions between PE Loopbacks (used as update-sources)

OSPF does **not** carry any customer/VRF routes — those are handled by MP-BGP.

| Router | OSPF Router-ID | Networks Advertised |
|--------|----------------|---------------------|
| R1C | 10.255.1.1 | 10.0.0.0/8 (area 0), 10.255.1.1/32 |
| R2C | 10.255.2.2 | 10.0.0.0/8 (area 0), 10.255.2.2/32 |
| R3C | 10.255.3.3 | 10.0.0.0/8 (area 0), 10.255.3.3/32 |
| R1E | 10.255.11.11 | 10.0.101.0/30, 10.255.11.11/32 |
| R2E | 10.255.12.12 | 10.0.102.0/30, 10.255.12.12/32 |
| R3E | 10.255.13.13 | 10.0.103.0/30, 10.255.13.13/32 |

---

## MPLS & LDP

All P-to-P core links and all P-to-PE links have `mpls ip` enabled. LDP is used as the label distribution protocol (`mpls label protocol ldp`). Each router anchors its LDP Router-ID to Loopback0 (`mpls ldp router-id Loopback0 force`), ensuring LDP sessions use stable, routable addresses.

**LDP neighborship matrix:**

| Session | Path |
|---------|------|
| R1C ↔ R2C | Gi0/0/0 / Gi0/0/0 |
| R1C ↔ R3C | Gi0/0/1 / Gi0/0/0 |
| R2C ↔ R3C | Gi0/0/1 / Gi0/0/1 |
| R1C ↔ R1E | Gi0/0/2 / Gi0/0/2 |
| R2C ↔ R2E | Gi0/0/2 / Gi0/0/2 |
| R3C ↔ R3E | Gi0/0/2 / Gi0/0/2 |

The P routers form full LDP adjacency with each other and with their paired PE. The PE routers only have one LDP neighbor each (their directly connected P router). Full label reachability for all Loopbacks is distributed across the entire MPLS domain via LDP + OSPF.

---

## VRF & L3VPN Design

The lab uses **MPLS L3VPN (RFC 4364)** with a hub-and-spoke + full-mesh hybrid model:

### Route Targets — VPN topology

| VRF | Export RT | Import RT(s) | Effect |
|-----|-----------|--------------|--------|
| CLIENT1 | 65000:1 | 65000:1, **65000:100** | Receives routes from own sites + DC_CENTRAL |
| CLIENT2 | 65000:2 | 65000:2, **65000:100** | Receives routes from own sites + DC_CENTRAL |
| CLIENT3 | 65000:3 | 65000:3, **65000:100** | Receives routes from own sites + DC_CENTRAL |
| CLIENT4 | 65000:4 | 65000:4, **65000:100** | Receives routes from own sites + DC_CENTRAL |
| CLIENT5 | 65000:5 | 65000:5, **65000:100** | Receives routes from own sites + DC_CENTRAL |
| DC_CENTRAL | 65000:100 | 65000:100, **65000:1–5** | Receives routes from ALL clients |

**Key observations:**
- Each client VRF is **isolated from all other client VRFs** — no client-to-client RT import. Clients cannot reach each other directly.
- Every client imports RT `65000:100`, so they all receive routes exported by `DC_CENTRAL`.
- `DC_CENTRAL` imports all five client RTs (`65000:1` through `65000:5`), so the data centre can reach every client.
- This is a **hub-and-spoke L3VPN design** where `R3E / DC_CENTRAL` is the hub and all client VRFs are spokes.

---

## BGP Design

### iBGP Full Mesh (PE Routers)

All three PE routers form a **full iBGP mesh** within AS 65000, exchanging VPNv4 prefixes:

```
R1E (10.255.11.11) ──────── R2E (10.255.12.12)
       │                          │
       └────────── R3E (10.255.13.13) ──────────┘
```

- All sessions use `update-source Loopback0`
- All sessions are activated in the `address-family vpnv4` with `send-community extended` (required to carry Route Targets)

### eBGP PE-to-CE Sessions (per VRF)

Each PE uses a separate eBGP session per VRF toward the CE/aggregation device:

| PE | VRF | PE IP | CE IP | CE AS |
|----|-----|-------|-------|-------|
| R1E | CLIENT1 | 172.16.1.1 | 172.16.1.2 | 65021 |
| R1E | CLIENT2 | 172.16.2.1 | 172.16.2.2 | 65021 |
| R2E | CLIENT3 | 172.16.3.1 | 172.16.3.2 | 65022 |
| R2E | CLIENT4 | 172.16.4.1 | 172.16.4.2 | 65022 |
| R3E | CLIENT5 | 172.16.5.1 | 172.16.5.2 | 65023 |
| R3E | DC_CENTRAL | 172.16.100.1 | 172.16.100.2 | 65023 |

SW_AGR1 uses **AS 65021** and peers with R1E for both CLIENT1 and CLIENT2 VRFs.

---

## Traffic Flow

### Client-to-DC_CENTRAL (spoke-to-hub)

This is the primary intended traffic path — all clients can reach the central data centre.

**Example: CLIENT1 (on SW_AGR1) → DC_CENTRAL (on R3E)**

```
[CLIENT1 LAN]
      │  (routed by SW_AGR1 VRF CLIENT1)
      ▼
SW_AGR1 Gi1/0/1 → R1E Gi0/0/0 (eBGP, AS65021 → AS65000, VRF CLIENT1)
      │
      ▼
R1E: Packet enters VRF CLIENT1. Destination matches DC_CENTRAL prefix
     learned via VPNv4 iBGP from R3E (next-hop 10.255.13.13).
     Label stack imposed: [LDP label to reach 10.255.13.13 | VPN label for DC_CENTRAL]
      │
      ▼
R1E Gi0/0/2 → R1C Gi0/0/2  (MPLS-labeled, inner label = VPN label)
      │
      ▼
R1C: Label lookup → swaps outer LDP label, forwards toward R3C or R2C
     (OSPF equal-cost paths available: R1C→R3C or R1C→R2C→R3C)
      │
      ▼
...P routers swap labels along LSP toward 10.255.13.13...
      │
      ▼
R3C Gi0/0/2 → R3E Gi0/0/2  (penultimate-hop pop or label swap)
      │
      ▼
R3E: Pops VPN label, identifies VRF DC_CENTRAL, routes packet to
     CE via Gi0/0/0.200 (VLAN 200) → 172.16.100.2
      │
      ▼
[DC_CENTRAL server / gateway at 172.16.100.2]
```

### DC_CENTRAL-to-Client (hub-to-spoke)

Return path is symmetric. DC_CENTRAL's BGP router (AS 65023) advertises its prefixes into the `DC_CENTRAL` VRF on R3E. R3E exports them with RT `65000:100`. All client VRFs on R1E and R2E import RT `65000:100`, so they install DC_CENTRAL routes. Return labeled packets travel R3E → R3C → (P core) → R1C/R2C → R1E/R2E → CE.

### Client-to-Client (BLOCKED by design)

Clients are **fully isolated**. CLIENT1 only imports RT `65000:1` and `65000:100`. It does **not** import `65000:2`, `65000:3`, etc. Therefore:
- No VPNv4 routes from other client VRFs are installed.
- Client-to-client traffic is dropped at the PE with no route match.
- This is intentional — clients share only the DC_CENTRAL hub.

### MPLS Label-Switched Path (LSP) Detail

Traffic between any two PEs traverses a label-switched path established by LDP over OSPF:

```
PE (ingress) → impose [VPN label + LDP transport label]
P router(s)  → swap outer LDP label only (P routers are VRF-unaware)
P router (penultimate) → PHP (Penultimate Hop Popping) removes transport label
PE (egress)  → only VPN label present; identifies VRF and forwards to CE
```

The triangular P-core (`R1C–R2C–R3C`) provides two equal-cost paths between any two PE sites, enabling load-sharing and redundancy. OSPF equal-cost multipath (ECMP) will distribute LDP transport LSPs over both paths where available.

---

## Inter-Client Connectivity Matrix

| | CLIENT1 | CLIENT2 | CLIENT3 | CLIENT4 | CLIENT5 | DC_CENTRAL |
|---|---|---|---|---|---|---|
| **CLIENT1** | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **CLIENT2** | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ |
| **CLIENT3** | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
| **CLIENT4** | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ |
| **CLIENT5** | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **DC_CENTRAL** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

✅ = reachable (RT import configured) | ❌ = isolated (no RT import)

---

*Documentation generated: 2026-03-12 | Repository: JakubSzarpak/Multi-Site-WAN-Cisco-Physical-Gear-Lab*
