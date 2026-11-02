# DC Server — Supermicro SYS-510T-M / VMware ESXi

Functional description of the server attached to the lab's data-centre switch: what it is, what runs on it, and how it relates to the operator network. Network-side facts come from the device configs in this repo.

---

## 1. Role in the lab

The Supermicro server is **not part of the operator's infrastructure**. From the network's point of view it is an external system, like a third-party server colocated in the operator's data centre. The operator delivers connectivity to it, and the owner runs everything inside.

Its virtual machines play the **outside world** for the customer sites: hosts on the public segment that branch offices can reach across the MPLS backbone. The server provides no service to the operator network itself. It does not route for it, assign addresses in it or take part in its control plane.

## 2. Hardware

| | |
|---|---|
| Model | Supermicro SuperServer SYS-510T-M (1U rack, single socket) |
| Motherboard | Supermicro X12STH-SYS |
| CPU | Intel Xeon E-2336: 6 cores / 12 threads |
| Memory | 64 GB DDR4-3200 ECC UDIMM (4 × 16 GB; platform maximum 128 GB) |
| Boot device | 256 GB M.2 NVMe (hypervisor only) |
| VM storage | 2 × 960 GB SATA SSD in front hot-swap bays, each a separate datastore |
| Network | 2 × 1 GbE RJ45 (Intel I210) + dedicated 1 GbE BMC port |
| Out-of-band | ASPEED AST2600 BMC: IPMI 2.0, Redfish, remote console and power control |
| Power | 1 × 350 W Platinum PSU, non-redundant |

The SSDs are deliberately not mirrored. The board has no hardware RAID controller, and ESXi does not support the chipset's software RAID. Each SSD is an independent datastore, and VM data is protected by being easy to rebuild rather than by redundancy (§7).

## 3. Hypervisor

- **VMware ESXi 8.0 U3**, standalone host, managed through the ESXi Host Client (web UI). There is no vCenter, so no HA, vMotion or clustering. With a single host they would have nothing to act on anyway.
- The hypervisor itself is reachable only on the private management hand-off. None of the VMs has a network path to it (§5).
- Current load: two small VMs on a 6-core / 64 GB host, leaving plenty of headroom for further test machines.

## 4. Network attachment

The server uses three physical links into SW_AGR3, one per function. Each lands on an untagged access port, so the switch alone decides which VRF the traffic belongs to. The hypervisor does no VLAN tagging.

```
   Customer sites (CLIENT1–5)
            │   MPLS L3VPN — IS-IS + SR-MPLS core, MP-BGP VPNv4
            ▼
          R3E   (PE, Site 3)
            │   802.1Q:  VLAN 200 → DC_CENTRAL,  VLAN 201 → DC_PUB
            ▼
         SW_AGR3   (operator hand-off: gateway for every segment)
      ┌─────┼──────────────────┐
  Gi1/0/20  Gi1/0/21        Gi1/0/23
  VLAN 500  VLAN 510        VLAN 600
      │       │                │
    LAN1     BMC              LAN2
      │       │                │
  ┌───┴───────┴────────────────┴───┐
  │  Supermicro SYS-510T-M — ESXi  │
  │   ESXi management   dhcp01     │
  │                     srv01      │
  └────────────────────────────────┘
```

| Server port | Switch port | VLAN / VRF | Subnet / gateway | Carries |
|---|---|---|---|---|
| LAN1 | Gi1/0/20 | 500 / DC_CENTRAL | 10.100.0.0/24 / .1 | ESXi management interface (10.100.0.10) |
| BMC | Gi1/0/21 | 510 / DC_CENTRAL | 10.100.10.0/24 / .1 | Out-of-band hardware management (10.100.10.10) |
| LAN2 | Gi1/0/23 | 600 / DC_PUB | 203.0.113.0/24 / .1 | All VM traffic |

- **Management and workload are split by VRF, not only by port.** The ESXi and BMC interfaces live in DC_CENTRAL, while the VMs live in DC_PUB. The two VRFs do not import each other's routes, so nothing on the VM network can reach the host's management.
- **The BMC is pinned to its dedicated port.** Supermicro's default BMC mode fails over onto LAN1 when the dedicated port is down. That would put the BMC into the wrong segment, so the failover is turned off.
- **ESXi networking mirrors the cabling:** one virtual switch per physical uplink. The management interface sits on the LAN1 switch. A single VM port group sits on the LAN2 switch, and no VM is attached to the management side.

## 5. Who can reach what

Reachability is defined by route-target policy on the operator's PE routers. The server's own design adds the rule that VMs never touch management.

| From ↓ / To → | ESXi management | BMC | VMs (public segment) |
|---|---|---|---|
| Customer sites (CLIENT1–5) | ✅ | ✅ | ✅ |
| VMs on this server | ❌ | ❌ | ✅ |

- Every customer VRF imports both DC route targets, so any branch office can use the VMs and can also administer the server.
- The public segment imports all customer routes, so the VMs can initiate traffic towards any site, e.g. for return-path tests.
- Customers still cannot reach each other. The server is a shared destination, not a transit point between them.

## 6. Virtual machines

| VM | OS | vCPU / RAM / disk | Address | Function |
|---|---|---|---|---|
| dhcp01 | Debian 12 | 1 / 1 GB / 16 GB | 203.0.113.10 (static) | Address assignment for VMs on this host |
| srv01 | Ubuntu Server 24.04 LTS | 2 / 4 GB / 40 GB | 203.0.113.20 (DHCP reservation) | Test endpoint for the customer sites |

### 6.1 dhcp01: address assignment for the server's VMs

- **Scope: the server's own VM network only.** DHCP requests are broadcasts and stop at the first router. The public segment's broadcast domain contains nothing but this host's VMs and the switch gateway, and no DHCP relay exists anywhere in the operator network. As a result, dhcp01 cannot assign addresses to customer LANs or operator links, and it does not try to.
- **What it hands out:** an address on 203.0.113.0/24 and the operator gateway 203.0.113.1 as default route.
- **Two kinds of lease:**
  - fixed reservations for named VMs, so srv01 always gets the same address;
  - a dynamic pool (203.0.113.100–199) for short-lived test VMs, which come up with working connectivity without any manual addressing.
- **Why it exists:** new VMs on the server need no hand configuration, while the operator side stays entirely static. That mirrors the real split of responsibility between a hosting customer and the carrier.

### 6.2 srv01: test endpoint

- A general-purpose Linux host that represents "a server on the Internet" for the branch offices.
- It is the fixed target for end-to-end checks from customer sites:
  - **reachability:** proves the route-target policy delivers DC_PUB routes to that customer's VRF;
  - **path tracing:** shows the hops across the operator backbone;
  - **throughput tests:** measure what a site actually gets through the PE, core and DC hand-off;
  - **remote login:** a realistic interactive session over the VPN.
- It runs no customer-facing application. Its value is being a stable, known destination when validating the network.

## 7. Operations and limitations

| Area | State |
|---|---|
| Availability | Single host, single PSU, one uplink per function, one path into the core (R3E → R3C). Any of these failing takes the server offline. |
| Storage protection | None at disk level (see §2). Both VMs are small and stateless apart from the DHCP lease database, so recovery means rebuilding them. |
| Backups | None. Acceptable for a lab system with no unique data. |
| Monitoring | Hardware health via the BMC; no central monitoring. |
| Management exposure | ESXi and the BMC are reachable from every customer site, a lab convenience so any branch can act as the admin workstation. In production they would sit in a dedicated management VRF that customers do not import. |

## 8. Roadmap

Plans recorded in the project history:

- **ASA-1 (Cisco ASA 5512-X) in front of the server:** stateful inspection of customer-to-server traffic, allowing only the required services.
- **Slican IP-PBX** on the private segment (switch port Gi1/0/22 already reserved).
- **Private NetFlow collector** in a VRF that customers cannot import.
