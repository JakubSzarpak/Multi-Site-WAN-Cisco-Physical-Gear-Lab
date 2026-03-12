# Multi-Site WAN — Cisco Physical Gear Lab

## Table of Contents
1. [Overview](#overview)
2. [Hardware & Software](#hardware--software)
3. [Network Topology](#network-topology)
4. [IP Addressing](#ip-addressing)
5. [Device Roles & Configuration Summary](#device-roles--configuration-summary)
   - [Core Routers (P Routers)](#core-routers-p-routers)
   - [PE Routers](#pe-routers)
   - [Aggregation Switches](#aggregation-switches)
6. [Routing Protocols](#routing-protocols)
7. [MPLS & LDP](#mpls--ldp)
8. [VRF & L3VPN Design](#vrf--l3vpn-design)
9. [BGP Design](#bgp-design)
10. [Traffic Flow](#traffic-flow)
11. [Inter-Client Connectivity Matrix](#inter-client-connectivity-matrix)
12. [Bugs Found & Fixes Applied](#bugs-found--fixes-applied)
13. [Client LAN Subnets](#client-lan-subnets)
14. [Future Roadmap — ASA Firewalls, VoIP & DC Services](#future-roadmap--asa-firewalls-voip--dc-services)

---

## Overview

This lab simulates a **service-provider / enterprise WAN** built on physical Cisco gear. It implements a full **MPLS L3VPN** backbone with three sites, each served by a dedicated PE (Provider Edge) router. A triangular core (P-router mesh) provides redundant label-switched paths across the backbone. Five customer VRFs (`CLIENT1`–`CLIENT5`) represent isolated branch sites, and a sixth VRF (`DC_CENTRAL`) acts as a shared Data-Centre hub that all clients can reach.

---

## Hardware & Software

| Device | Model | OS | Role |
|--------|-------|----|------|
| R1C, R2C, R3C | Cisco ISR 4331 | IOS-XE 16.x (Fuji) – 17.x (Bengaluru) | MPLS Core (P routers) |
| R1E, R2E, R3E | Cisco ISR 4331 | IOS-XE 16.x (Fuji) – 17.x (Bengaluru) | Provider Edge (PE routers) |
| SW_AGR1 | Cisco Catalyst 3750-E | IOS (IP Services) | Site 1 Aggregation (CE) |
| SW_AGR2 | Cisco Catalyst 3750-E | IOS (IP Services) | Site 2 Aggregation (CE) |
| SW_AGR3 | Cisco Catalyst 3750-E | IOS (IP Services) | Site 3 Aggregation + DC (CE) |

**Available but not yet deployed:**

| Device | Model | Planned Role |
|--------|-------|--------------|
| ASA-1 | Cisco ASA 5512-X | DC Firewall (in front of Supermicro server) |
| ASA-2 | Cisco ASA 5512-X | Site perimeter / inter-VRF firewall |
| ASA-3 | Cisco ASA 5512-X | Site perimeter / inter-VRF firewall |
| Supermicro Server | Supermicro | DC server (services for all clients) |
| Slican SIP Server | Slican | VoIP PBX (SIP trunks + phones) |

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
| PE-AGR | R1E | Gi0/0/0 (trunk) | — | SW_AGR1 | Gi1/0/1 (trunk) | VLANs 101,102 |
| PE-AGR | R2E | Gi0/0/0 (trunk) | — | SW_AGR2 | Gi1/0/1 (trunk) | VLANs 103,104 |
| PE-AGR | R3E | Gi0/0/0 (trunk) | — | SW_AGR3 | Gi1/0/1 (trunk) | VLANs 105,200 |

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

### Aggregation Switches — CE-side (VRF-Lite, per-VLAN SVIs)

| Switch | VRF | PE-link SVI | SVI IP | LAN SVI | LAN Gateway |
|--------|-----|------------|--------|---------|-------------|
| SW_AGR1 | CLIENT1 | Vlan101 | 172.16.1.2/30 | Vlan301 | 192.168.1.1/24 |
| SW_AGR1 | CLIENT2 | Vlan102 | 172.16.2.2/30 | Vlan302 | 192.168.2.1/24 |
| SW_AGR2 | CLIENT3 | Vlan103 | 172.16.3.2/30 | Vlan303 | 192.168.3.1/24 |
| SW_AGR2 | CLIENT4 | Vlan104 | 172.16.4.2/30 | Vlan304 | 192.168.4.1/24 |
| SW_AGR3 | CLIENT5 | Vlan105 | 172.16.5.2/30 | Vlan305 | 192.168.5.1/24 |
| SW_AGR3 | DC_CENTRAL | Vlan200 | 172.16.100.2/30 | Vlan500/510 | 10.100.0.1/24, 10.100.10.1/24 |

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

### Aggregation Switches

All three Catalyst 3750-E switches implement **VRF-Lite** with **eBGP** toward their paired PE router. Each uses an 802.1Q trunk uplink carrying per-client VLANs, with SVIs providing the L3 BGP peering addresses and client LAN gateways.

#### SW_AGR1 — Site 1 (AS 65021 → R1E)

| VRF | PE-link VLAN | SVI IP (BGP peer) | LAN VLAN | LAN Gateway |
|-----|----|---|---|---|
| CLIENT1 | 101 | 172.16.1.2/30 | 301 | 192.168.1.1/24 |
| CLIENT2 | 102 | 172.16.2.2/30 | 302 | 192.168.2.1/24 |

#### SW_AGR2 — Site 2 (AS 65022 → R2E)

| VRF | PE-link VLAN | SVI IP (BGP peer) | LAN VLAN | LAN Gateway |
|-----|----|---|---|---|
| CLIENT3 | 103 | 172.16.3.2/30 | 303 | 192.168.3.1/24 |
| CLIENT4 | 104 | 172.16.4.2/30 | 304 | 192.168.4.1/24 |

#### SW_AGR3 — Site 3 + DC (AS 65023 → R3E)

| VRF | PE-link VLAN | SVI IP (BGP peer) | LAN VLAN | LAN Gateway |
|-----|----|---|---|---|
| CLIENT5 | 105 | 172.16.5.2/30 | 305 | 192.168.5.1/24 |
| DC_CENTRAL | 200 | 172.16.100.2/30 | 500 (servers) / 510 (mgmt) | 10.100.0.1/24, 10.100.10.1/24 |

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

## Bugs Found & Fixes Applied

The following issues were identified during config audit and corrected:

### 🔴 Critical (would prevent routing from working)

| # | Device | Issue | Fix |
|---|--------|-------|-----|
| 1 | **R1E, R2E, R3E** | VRF `route-target` statements placed **outside** `address-family ipv4`. On IOS-XE `vrf definition`, RT import/export are ignored unless inside the address-family block. **No VPN routes would be imported/exported.** | Moved all `route-target export/import` lines inside `address-family ipv4 … exit-address-family` |
| 2 | **R1E, R2E, R3E** | `vrf definition` blocks appeared **after** the interfaces using `vrf forwarding`. IOS-XE requires the VRF to exist before an interface can reference it — config would be rejected on paste. | Moved all `vrf definition` blocks to the **top** of the config, before any interface |
| 3 | **SW_AGR1** | Uplink `Gi1/0/1` had IP `172.16.1.1/30` — **identical to R1E Gi0/0/0.101**. IP conflict = duplicate address detection, no adjacency. | Converted uplink to 802.1Q trunk; moved L3 to SVIs with correct IP `172.16.1.2` |
| 4 | **SW_AGR1** | BGP peered with `172.16.1.1` in **both** CLIENT1 and CLIENT2 VRFs. `172.16.1.1` was SW_AGR1's own address (looped to self). CLIENT2 should peer with `172.16.2.1`. | CLIENT1 peers with `172.16.1.1` (R1E .101 sub-if), CLIENT2 peers with `172.16.2.1` (R1E .102 sub-if) — each via correct SVI/VLAN |
| 5 | **SW_AGR1** | Uplink was a **plain routed port** but R1E sends **802.1Q-tagged** frames on sub-interfaces. Frames with dot1Q tags arriving on an access/routed port would be dropped. | Changed `Gi1/0/1` to `switchport mode trunk` with `allowed vlan 101,102` |

### 🟡 Functional (routing would work but with gaps)

| # | Device | Issue | Fix |
|---|--------|-------|-----|
| 6 | **SW_AGR1** | No BGP `network` statements — client LAN subnets were never advertised into BGP. Clients could not be reached from DC_CENTRAL. | Added `network 192.168.x.0 mask 255.255.255.0` in each VRF address-family |
| 7 | **SW_AGR1** | Client-facing ports (`Gi1/0/10`, `Gi1/0/11`) were VRF routed ports with /30 links but no LAN gateway for actual end-users. | Replaced with switchport access ports + SVI-based gateways on /24 LAN VLANs |
| 8 | **R1E, R2E, R3E** | `Gi0/0/0` parent interface lacked `no ip address` — could cause ambiguity for IOS-XE sub-interface routing. | Added explicit `no ip address` on all PE trunk parent interfaces |

### 🟢 Cosmetic / Hardening

| # | Device | Issue | Fix |
|---|--------|-------|-----|
| 9 | All switches | No `vtp mode transparent`, no `spanning-tree mode`, no console/VTY hardening | Added `vtp mode transparent`, `rapid-pvst`, `portfast` on access ports, VTY SSH |
| 10 | All switches | Missing VLAN definitions | Added all VLANs with descriptive names |

---

## Client LAN Subnets

Standardized /24 LAN subnets for each client, advertised via eBGP into the MPLS backbone:

| Client | LAN Subnet | Gateway (on AGR switch) | VLAN | Switch |
|--------|-----------|------------------------|------|--------|
| CLIENT1 | 192.168.1.0/24 | 192.168.1.1 | 301 | SW_AGR1 |
| CLIENT2 | 192.168.2.0/24 | 192.168.2.1 | 302 | SW_AGR1 |
| CLIENT3 | 192.168.3.0/24 | 192.168.3.1 | 303 | SW_AGR2 |
| CLIENT4 | 192.168.4.0/24 | 192.168.4.1 | 304 | SW_AGR2 |
| CLIENT5 | 192.168.5.0/24 | 192.168.5.1 | 305 | SW_AGR3 |
| DC Servers | 10.100.0.0/24 | 10.100.0.1 | 500 | SW_AGR3 |
| DC Mgmt | 10.100.10.0/24 | 10.100.10.1 | 510 | SW_AGR3 |

---

## Future Roadmap — ASA Firewalls, VoIP & DC Services

### Phase 1 — Current (Completed ✅)
- MPLS L3VPN backbone operational
- All 3 PE routers with correct VRF/RT/BGP config
- All 3 aggregation switches ready to plug in
- Hub-and-spoke VPN: all clients → DC_CENTRAL

### Phase 2 — DC Firewall (ASA-1 between SW_AGR3 and Supermicro)

Place the first ASA 5512-X as a **transparent or routed firewall** between SW_AGR3 and the Supermicro server:

```
SW_AGR3 VLAN 500 ──── ASA-1 inside ──── Supermicro Server
                      ASA-1 outside ─── (stays in DC_CENTRAL VRF)
```

**Recommended config approach:**
- ASA-1 in **routed mode** with `inside` (10.100.0.0/24 gateway) and `outside` toward SW_AGR3
- Inspect all traffic from clients to server; permit only required services (HTTPS, RDP, SSH, SIP, etc.)
- NAT not required (all private addressing within MPLS VPN)
- Security levels: inside=100, outside=0

### Phase 3 — VoIP (Slican SIP Server in DC)

Deploy Slican VoIP PBX on the DC server VLAN. Options:

**Option A — Shared DC_CENTRAL VRF (simplest):**
- Slican on VLAN 500 (10.100.0.x) — phones at client sites reach it via DC_CENTRAL hub
- QoS markings (DSCP EF for voice, AF31 for signalling) must be configured on all PE and P interfaces
- Consider adding a dedicated **VOICE VLAN** on each AGR switch for phone ports

**Option B — Dedicated VOICE VRF (advanced, better isolation):**
- New VRF `VOICE` on each PE with its own RT (e.g., `65000:10`)
- Slican server imports `65000:10`; each client VOICE VRF exports/imports `65000:10`
- Complete separation of voice and data traffic

**QoS policy needed on all routers (both options):**
```
class-map match-any VOICE
 match dscp ef
class-map match-any VOICE-SIGNALING
 match dscp cs3  af31
policy-map WAN-QOS
 class VOICE
  priority percent 20
 class VOICE-SIGNALING
  bandwidth percent 5
 class class-default
  fair-queue
```

### Phase 4 — Site Perimeter Firewalls (ASA-2, ASA-3)

Place ASA-2 and ASA-3 at two of the three sites between the AGR switch and client LANs:

```
Client LAN ──── ASA-2 ──── SW_AGR1 ──── R1E ──── MPLS Core
Client LAN ──── ASA-3 ──── SW_AGR2 ──── R2E ──── MPLS Core
```

- Each ASA runs in routed mode, one context per client VRF (multi-context if needed)
- Provides stateful inspection, URL filtering, and IPS at the site edge
- ASA peers eBGP with the AGR switch or uses static routes

### Suggested IP Plan for ASA Integration

| ASA | Location | Inside | Outside | VRF |
|-----|----------|--------|---------|-----|
| ASA-1 | DC (SW_AGR3 → Server) | 10.100.0.254/24 | 10.100.0.1/24 (SW_AGR3) | DC_CENTRAL |
| ASA-2 | Site 1 (LAN → SW_AGR1) | 192.168.1.254/24 | transit /30 to SW_AGR1 | CLIENT1/CLIENT2 |
| ASA-3 | Site 2 (LAN → SW_AGR2) | 192.168.3.254/24 | transit /30 to SW_AGR2 | CLIENT3/CLIENT4 |

---

*Documentation updated: 2026-03-12 | Repository: JakubSzarpak/Multi-Site-WAN-Cisco-Physical-Gear-Lab*
