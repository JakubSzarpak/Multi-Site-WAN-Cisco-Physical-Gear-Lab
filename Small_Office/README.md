# Small Office — Branch Network for CLIENT1

Sample design of a small, fully redundant branch office connected to the lab's MPLS L3VPN as customer **CLIENT1**. It shows what sits behind the operator hand-off:

- a pair of firewalls;
- a redundant routed core;
- floor and room switches down to wall outlets, desk phones and Wi-Fi.

Hardware is chosen from gear typically found in a school Cisco lab (Networking Academy), while still supporting every feature the design relies on.

---

## 1. At a glance

| | |
|---|---|
| Customer | CLIENT1 (VRF on PE R1E, hand-off on SW_AGR1) |
| Office address space | 10.1.0.0/16, advertised to the operator as one summary |
| Operator hand-off | 2 links, VLAN 301, 192.168.1.0/24, OSPF |
| Edge | 2 × Cisco ASA 5512-X, active/standby stateful failover |
| Core | 2 × Catalyst 3650, inter-VLAN routing, VRRP gateways |
| Floor | 1 × Catalyst 2960-X per floor, uplinked to both cores |
| Room | 1 × Catalyst 2960-CX per room, PoE+ |
| Wireless | Catalyst 9115AXI access points with embedded controller |
| Building | 2 floors, 4 rooms |

## 2. Does the operator side allow two links?

**Routing: yes, without changes.** On the operator side, VLAN 301 is an OSPF segment inside VRF CLIENT1. The aggregation switch:

- accepts any OSPF neighbour on that segment;
- sends it a default route plus the routes to the data centre;
- redistributes the customer's routes into BGP, so the office becomes reachable from the DC across the backbone.

There is one catch. The operator only carries **internal** OSPF routes into BGP. Anything the customer injects as an external route (redistributed connected or static networks) stays local and never reaches the DC. The office therefore hands its prefixes over as an inter-area summary from the firewalls (§5).

**Physically: one extra port is needed.** CLIENT1 has a single access port in VLAN 301 on SW_AGR1 (Gi1/0/10). The second firewall needs one more port in the same VLAN, marked (*) below. That is the only operator-side change this design requires, and it is not part of this folder.

**What it protects against.** Both links land on the same aggregation switch (SW_AGR1), PE (R1E) and core router (R1C). They protect against a firewall, cable or port failure, not against an operator node failure. True dual-homing would need the CLIENT1 VRF on a second PE.

## 3. Hardware

| Role | Qty | Model | Why this one |
|---|---|---|---|
| Firewall | 2 | Cisco ASA 5512-X, ASA 9.12, Security Plus licence | Already in the lab inventory (ASA-2, ASA-3 in the roadmap). Stateful active/standby failover, which on this model requires the Security Plus licence; OSPF; VLAN sub-interfaces. 9.12 is the last release for this model. |
| Core | 2 | Catalyst 3650-24TS-E (IP Services), IOS-XE 16.12 | Full Layer 3: OSPF, VRRPv3 with object tracking, DHCP server, ACLs. 24 × 1G copper + 4 × 1G SFP. |
| Floor | 2 | Catalyst 2960-X-24TS-L | Rapid PVST+, LACP, DHCP snooping; 4 SFP ports for fibre uplinks. |
| Room | 4 | Catalyst 2960-CX-8PC-L | Compact and fanless, so it can live in the room. 8 × PoE+ with a 124 W budget, 2 copper + 2 SFP uplinks. |
| Wi-Fi | 4 | Catalyst 9115AXI | Wi-Fi 6, powered over PoE+, Embedded Wireless Controller: no separate WLC needed. |
| Phones | per desk | SIP desk phones, PoE, LLDP-MED | Register to the planned Slican PBX in the data centre. |

**The two 3650s are deliberately not stacked.** A stack is one control plane, so a bad upgrade or a stack fault takes both members down at once. Two independent switches with VRRP survive that, and they show first-hop redundancy, which is the point of the design.

## 4. Topology

```
      Operator: SW_AGR1 · VRF CLIENT1 · VLAN 301 · 192.168.1.0/24
              Gi1/0/10                          Gi1/0/12 (*)
                  │                                  │
            ┌─────┴─────┐                      ┌─────┴─────┐
            │    FW-A   ├───failover + state───┤    FW-B   │
            │ ASA 5512-X│                      │ ASA 5512-X│
            └─────┬─────┘                      └─────┬─────┘
                  │  inside trunk: VLAN 40, 900      │
            ┌─────┴─────┐                      ┌─────┴─────┐
            │   CORE-1  ╞════LACP Po1 2×1G═════╡   CORE-2  │
            │  Cat 3650 │   VRRP · STP root    │  Cat 3650 │
            └─────┬──┬──┘                      └──┬──┬─────┘
                  │  │                            │  │
                  │  └─────────────┐              │  │
                  │                │              │  │  fibre
                  │  ┌─────────────┼──────────────┘  │  uplinks
                  │  │             └──────────────┐  │
                  │  │                            │  │
            ┌─────┴──┴──┐                      ┌──┴──┴─────┐
            │   FL0-SW  │                      │   FL1-SW  │
            │ Cat 2960-X│                      │ Cat 2960-X│
            └──┬─────┬──┘                      └──┬─────┬──┘
               │     │                            │     │
           ┌───┘     └───┐                    ┌───┘     └───┐
           │ LACP   LACP │                    │ LACP   LACP │
       ┌───┴───┐     ┌───┴───┐            ┌───┴───┐     ┌───┴───┐
       │ RM-0.1│     │ RM-0.2│            │ RM-1.1│     │ RM-1.2│
       │2960-CX│     │2960-CX│            │2960-CX│     │2960-CX│
       └───────┘     └───────┘            └───────┘     └───────┘
        wall outlets · desk phones · PCs · printers · Wi-Fi APs

 (*) second operator port: must be added on SW_AGR1, see §2
```

| Location | Contents |
|---|---|
| Floor 0, comms rack | FW-A, FW-B, CORE-1, CORE-2, FL0-SW |
| Floor 0, room 0.1 | Reception: RM-0.1 |
| Floor 0, room 0.2 | Open office: RM-0.2 |
| Floor 1, floor cabinet | FL1-SW |
| Floor 1, room 1.1 | Office: RM-1.1 |
| Floor 1, room 1.2 | Meeting room: RM-1.2 |

### Cabling

| Link | Between | Medium | Carries |
|---|---|---|---|
| Operator A | SW_AGR1 Gi1/0/10 ↔ FW-A Gi0/0 | copper | VLAN 301 (untagged) |
| Operator B | SW_AGR1 Gi1/0/12 (*) ↔ FW-B Gi0/0 | copper | VLAN 301 (untagged) |
| Failover | FW-A Gi0/2 ↔ FW-B Gi0/2 | copper, direct | failover heartbeat + session state |
| Firewall inside | FW-A Gi0/1 ↔ CORE-1 Gi1/0/1, FW-B Gi0/1 ↔ CORE-2 Gi1/0/1 | copper | trunk: VLAN 40, 900 |
| Core interlink | CORE-1 Gi1/0/23–24 ↔ CORE-2 Gi1/0/23–24 | copper | LACP trunk, all VLANs |
| Floor uplinks | CORE-1 Gi1/1/1 → FL0-SW Gi1/0/25, CORE-1 Gi1/1/2 → FL1-SW Gi1/0/25, CORE-2 Gi1/1/1 → FL0-SW Gi1/0/26, CORE-2 Gi1/1/2 → FL1-SW Gi1/0/26 | 1000BASE-SX multimode fibre | trunk: office VLANs |
| Room uplinks | each room switch: both copper uplinks → its floor switch | copper | LACP trunk: office VLANs |
| Edge | room switch PoE+ ports → wall outlets, APs, printers | copper | per port type (§8) |

## 5. Routing

- **Outside (firewalls ↔ operator):** OSPF area 0 with the operator's switch. The firewalls learn the default route and the DC routes, and advertise the whole office as one summary, 10.1.0.0/16.
- **Inside (firewalls ↔ cores):** OSPF area 1 over the transit VLAN 900. The firewall pair is the area border router between the two areas.
- **Area 1 is totally stubby.** The cores learn only the office's own subnets plus a default route towards the firewalls. Office switches never see operator routes, and the operator never sees the office's internal structure. That is the same abstraction the operator applies to its own customers.
- **The summary reaches the DC.** It is an inter-area route, so the operator treats it as internal OSPF and carries it into BGP. No operator change is needed for routing (§2).
- **Failover.** Only the active firewall speaks OSPF. On failover, the standby unit takes over the same addresses, and graceful restart keeps the neighbours forwarding while adjacencies re-form.
- **Shared transit VLAN.** VLAN 900 spans both cores through the interlink. Whichever firewall is active is reachable from both cores without any routing change.

## 6. VLANs and addressing

| VLAN | Name | Subnet | Gateway | Active gateway | DHCP |
|---|---|---|---|---|---|
| 10 | STAFF | 10.1.10.0/24 | .1 (VRRP) | CORE-1 | cores, split scope |
| 20 | VOICE | 10.1.20.0/24 | .1 (VRRP) | CORE-2 | cores, split scope |
| 30 | DEVICES | 10.1.30.0/24 | .1 (VRRP) | CORE-1 | cores, split scope, reservations for printers |
| 40 | GUEST | 10.1.40.0/24 | .1 | firewall pair | firewall |
| 99 | MGMT | 10.1.99.0/24 | .1 (VRRP) | CORE-2 | none (static) |
| 900 | TRANSIT | 10.1.255.0/29 | — | FW .1 (standby .2), CORE-1 .4, CORE-2 .5 | none (static) |
| — | Operator hand-off | 192.168.1.0/24 | operator .1 | FW .2 (standby .3) | none (static) |

- **Gateway addresses.** In every VRRP VLAN, CORE-1 owns .2, CORE-2 owns .3, and .1 is the shared virtual gateway.
- **Split-scope DHCP.** Both cores serve every user VLAN, each from its own half of the range (CORE-1 the lower half, CORE-2 the upper). Either core alone can keep handing out leases, and the two can never assign the same address.
- **Unique addressing.** The DC VRFs import routes from every customer, so office addresses must be unique across the whole VPN. Here, 10.1.0.0/16 follows the scheme 10.*customer number*.0.0/16.

## 7. First-hop redundancy (VRRP)

- **One virtual gateway per VLAN.** Each user VLAN has a gateway (.1) shared by both cores with VRRPv3. Hosts always use .1 and never notice which core answers.
- **Load is split.** CORE-1 is master for STAFF and DEVICES, CORE-2 for VOICE and MGMT, and each is backup for the other's VLANs.
- **STP root follows the VRRP master.** In every VLAN, the active gateway is also the Rapid PVST+ root bridge, and the other core is the secondary root. Layer-2 frames take the direct path to the active gateway instead of crossing the core interlink.
- **Tracking.** Each core tracks its route towards the firewalls. A core that loses its way out lowers its VRRP priority and hands its VLANs over, so hosts are never sent to a gateway that cannot forward.
- **Preemption with a delay.** A recovered core takes its VLANs back only after its routing has converged, not the moment it boots.

## 8. Switching layers

**Core (CORE-1, CORE-2)**
- Routes between office VLANs, runs VRRP and DHCP, and enforces inter-VLAN policy (§10).
- Carries the guest VLAN at Layer 2 only, with no gateway on the cores.
- Root guard on floor-facing ports, so no downstream switch can ever take over as root bridge.

**Floor (FL0-SW, FL1-SW)**
- Pure Layer 2, with one uplink to each core.
- Rapid PVST+ forwards each VLAN on the uplink towards that VLAN's root. Both uplinks carry traffic in normal operation, and either one can carry everything after a failure.

**Room (RM-0.1 … RM-1.2)**
- Pure Layer 2 at the edge, with a two-link LACP bundle to the floor switch. Losing one cable costs bandwidth, not connectivity.
- Edge port types:

| Port type | Behaviour |
|---|---|
| Wall outlet (phone + PC) | Phone joins VOICE, tagged; the voice VLAN is advertised via LLDP-MED. The PC behind the phone's pass-through port lands in STAFF, untagged. |
| Access point | Trunk: MGMT untagged for the AP itself, STAFF and GUEST tagged for the SSIDs. |
| Printer | DEVICES VLAN. |

- Protection on every user port:
  - **PortFast + BPDU Guard:** a looped cable or a rogue switch shuts the port instead of disturbing STP;
  - **DHCP snooping:** only the uplinks may answer DHCP, so a rogue DHCP server on a desk is ignored;
  - **Dynamic ARP Inspection:** built on the snooping table, blocks ARP spoofing;
  - **port security:** limits the number of MAC addresses per outlet;
  - **storm control:** caps broadcast floods.
- **PoE budget.** One Wi-Fi 6 AP plus a handful of desk phones per room fits well within the 124 W of a 2960-CX.

## 9. Wireless

- **Four APs, one controller.** There are 4 × Catalyst 9115AXI, one per room. One AP runs the Embedded Wireless Controller for all four, so configuration and RF management live in one place without a hardware controller.

| SSID | VLAN | Security |
|---|---|---|
| OFFICE | STAFF (10) | WPA3/WPA2-Personal; there is no RADIUS server in the lab |
| GUEST | GUEST (40) | WPA2-Personal with client isolation: guests cannot see each other |

- **Local switching.** Each AP switches client traffic straight onto its room switch (FlexConnect local switching). Traffic never detours through the controller AP and keeps flowing if the controller role moves to another AP.

## 10. Security policy

There are two enforcement points:
- **the firewall pair**, for everything entering or leaving the office;
- **ACLs on the cores**, for traffic between VLANs inside the office.

| Source | Allowed to reach |
|---|---|
| STAFF | DEVICES (printing), DC_PUB (simulated Internet), business services in DC_CENTRAL |
| VOICE | The PBX in DC_CENTRAL only |
| DEVICES | Nothing on its own initiative; replies only |
| GUEST | DC_PUB only |
| MGMT | Everything, for administration |
| Operator side (DC) | Replies to sessions opened from the office; DC_CENTRAL monitoring may reach MGMT |

- **The guest VLAN is never routed inside the office.** It passes the cores at Layer 2 straight to the firewall, which is its gateway and DHCP server. Guests get the simulated Internet and nothing else.
- **No NAT.** Office addresses are unique within the VPN, so traffic is routed end-to-end and every log shows real source addresses.

## 11. Failure scenarios

| Failure | What happens | User impact |
|---|---|---|
| Active firewall | Standby takes over the same addresses; the session table was already replicated over the failover link | Seconds; open sessions survive |
| Operator link or port in use | Interface monitoring fails the pair over to the other firewall and its link | Seconds; open sessions survive |
| CORE-1 | CORE-2 becomes VRRP master and STP root for all VLANs. If FW-A was active, its inside link drops and the pair fails over to FW-B. | About 3 s with default VRRP timers |
| One core interlink cable | LACP continues on the remaining link | None |
| One floor uplink | Rapid PVST+ moves the affected VLANs to the other uplink | Sub-second to a few seconds |
| One room uplink cable | LACP continues on the remaining link | None |
| Room switch | That room is offline | Accepted: one switch per room |
| Floor switch | That floor's rooms are offline | Accepted: one switch per floor |
| SW_AGR1, R1E or R1C (operator) | Office is cut off from the VPN | Accepted: operator single-homing (§2) |

## 12. Management

- **Management addressing:**
  - every switch has an address in MGMT (VLAN 99);
  - the APs take their management address there too (untagged on their trunk);
  - the firewalls are managed on their inside addresses.
- **Access.** Devices accept SSH only, and only from the MGMT VLAN.
- **Not yet in place.** Central time, logging and configuration backup are natural services for the data centre; none exist yet.
