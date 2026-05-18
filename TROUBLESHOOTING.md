# Troubleshooting Guide / Przewodnik Diagnostyczny

Compact field-debug reference for the Multi-Site WAN lab.
Configs flashed from `USB_Ready_cfg/*.cfg` (or the matching `.txt`
in `PE_Routers/` and `Aggregation_Switches/`).

---

## 1. Topology cheat sheet

```
       R1_C  ───── R2_C  ───── R3_C            ← IS-IS L2 / SR-MPLS core
        │  X       │  X       │                  (R1_C + R2_C are RRs;
        │          │          │                   R3_C is plain P)
       R1_E      R2_E       R3_E               ← PE (ISR 4331)
        │          │          │
       SW_AGR1   SW_AGR2    SW_AGR3            ← Aggregation (3750-E)
        │          │          │
   CLIENT1-20   CLIENT21-40   DC_PUB
   SHARED_101   SHARED_102    DC_CENTRAL
   (DC_PUB only) (both DCs)
```

- IS-IS Level-2, NET `49.0001.0102.5501.XXXX.00`, area `49.0001`.
- SR-MPLS prefix-SID indexes: R1_E=110, R2_E=120, R3_E=130 (core: 10/20/30).
- iBGP VPNv4: every PE peers to RR1 (`10.255.1.1`) and RR2 (`10.255.2.2`).
- eBGP per-VRF: PE (AS 65000) ↔ switch (AS 65021 / 65022 / 65023).

---

## 2. Tenant quick-lookup tables

### Site 1 — R1_E + SW_AGR1 (DC_PUB only, no Slican)

| Tenant | RD/RT | PE-link VLAN | LAN VLAN | PE-CE /30 | LAN /24 | Access port |
|---|---|---|---|---|---|---|
| CLIENT_N (1..20) | `65000:N` | `100+N` | `200+N` | `172.16.N.0/30` | `192.168.N.0/24` | `Gi1/0/(49-2N)` |
| SHARED_101 outlet K (1..20) | `65000:101` | 121 (single PE-link) | `250+K` | `172.16.101.0/30` | `192.168.(100+K).0/24` | `Gi1/0/(50-2K)` |

Examples: CLIENT3 → VLAN 103/203, 172.16.3.0/30, 192.168.3.0/24, port Gi1/0/43.
SHARED_101 outlet 7 → VLAN 257, 192.168.107.0/24, port Gi1/0/36.

### Site 2 — R2_E + SW_AGR2 (both DCs)

| Tenant | RD/RT | PE-link VLAN | LAN VLAN | PE-CE /30 | LAN /24 | Access port |
|---|---|---|---|---|---|---|
| CLIENT_N (21..40) | `65000:N` | `100+N` | `200+N` | `172.16.N.0/30` | `192.168.N.0/24` | `Gi1/0/(49-2(N-20))` |
| SHARED_102 outlet K (1..20) | `65000:102` | 141 (single PE-link) | `270+K` | `172.16.102.0/30` | `192.168.(120+K).0/24` | `Gi1/0/(50-2K)` |

Examples: CLIENT25 → VLAN 125/225, 172.16.25.0/30, 192.168.25.0/24, port Gi1/0/39.
SHARED_102 outlet 3 → VLAN 273, 192.168.123.0/24, port Gi1/0/44.

### Site 3 — R3_E + SW_AGR3 (DC hub)

| VRF | RD/RT | PE-link VLAN | LAN VLAN(s) | PE-CE /30 | LAN /24 | Server port |
|---|---|---|---|---|---|---|
| DC_PUB | `65000:99` | 201 | 600 | `172.16.199.0/30` | `203.0.113.0/24` | Gi1/0/23 (Supermicro pub) |
| DC_CENTRAL | `65000:100` | 200 | 500 + 510 | `172.16.100.0/30` | `10.100.0.0/24`, `10.100.10.0/24` | Gi1/0/20-21 (Supermicro mgmt), Gi1/0/22 (Slican) |

### Community RT scheme (R3_E side only)

| RT | Meaning | Exported by | Imported by |
|---|---|---|---|
| `65000:1000` | TO_DC_PUB | every tenant in the lab | DC_PUB |
| `65000:2000` | TO_DC_CENTRAL | only Site 2 tenants (CLIENT21-40 + SHARED_102) | DC_CENTRAL |

PE-CE convention: PE side = `.1`, switch side = `.2` on every /30.

---

## 3. Post-flash verification — top-down

Run this checklist on every device after flashing. Each step depends
on the previous one — don't move down until the layer above is green.

### Layer 1: physical + L2

```
show interfaces description | exclude admin
show interfaces status                          (3750-E only)
show interfaces trunk                           (3750-E only)
show cdp neighbors                              (sanity: who is on the other end)
```

Look for: every `TRUNK-TO-*` and `TO-*` shows `up/up`. Trunks should
show `dot1q` encapsulation and the right `allowed vlan` range
(101-121 on SW_AGR1; 121-141 on SW_AGR2; 200,201 on SW_AGR3).

### Layer 2: IS-IS backbone (PE + P only)

```
show isis neighbors
show isis database level-2
show ip route isis
show isis fast-reroute summary
```

Expected: 1 neighbor per IS-IS-enabled interface, all `L2`, state `UP`.
Every PE Loopback (10.255.11.11/12.12/13.13) and every P Loopback
should appear as `i L2` in the routing table.

### Layer 3: SR-MPLS data plane

```
show segment-routing mpls connected-prefix-sid-map ipv4
show mpls forwarding-table
show ip cef 10.255.13.13                        (example: path to R3_E)
```

Expected: prefix-SID labels installed for every PE Loopback. The CEF
output for a remote PE Loopback should show `mpls label N` outgoing.

### Layer 4: iBGP VPNv4 to RR

```
show bgp vpnv4 unicast all summary | include 10.255
show bgp vpnv4 unicast all neighbors 10.255.1.1 advertised-routes | count
show bgp vpnv4 unicast all neighbors 10.255.1.1 received-routes | count
```

Expected: both RR sessions `Established`, `State/PfxRcd` is a number
(not `Active` or `Idle`). On R1_E you should receive ~21 prefixes
from each RR (everything advertised by R2_E and R3_E).

### Layer 5: eBGP per-VRF (PE ↔ CE)

On a PE:
```
show bgp vpnv4 unicast vrf CLIENT3 summary
show bgp vpnv4 unicast vrf CLIENT3
```

On the switch:
```
show ip bgp ipv4 unicast vrf CLIENT3 summary
show ip route vrf CLIENT3
```

Expected: state `Established`, prefix count matches what the switch
is advertising (CLIENT3 → just `192.168.3.0/24`).

### Layer 6: OSPF VRF-Lite on switch

```
show ip ospf 3 neighbor                         (CLIENT3 example)
show ip route vrf CLIENT3 ospf
show ip ospf 3 database
```

Customer router (when plugged in) should appear as a neighbor,
state `FULL`. The default route (`0.0.0.0/0`) should be
`O*E2` in the CR's routing table.

### Layer 7: forwarding (end-to-end)

From a customer router:
```
ping vrf <none-on-CR> 203.0.113.10              (DC_PUB target)
traceroute 203.0.113.10
```

Traceroute should show 3-4 hops: CR → switch SVI → R*_E → R3_E → DC_PUB target.

---

## 4. Symptom → likely cause → fix

### "BGP eBGP session won't come up between PE and switch"

| Check | Command | Expected |
|---|---|---|
| Layer 1 trunk up? | `show int Gi0/0/0` on PE | up/up |
| VLAN allowed on switch trunk? | `show int trunk` on switch | VLAN N in allowed list |
| Both sides in same VRF? | `show vrf` on PE, `show ip vrf` on switch | tenant VRF present on both |
| /30 reachable both ways? | `ping vrf CLIENT3 172.16.3.2` from PE | success |
| Remote AS correct? | grep `remote-as` for the neighbor | PE says 65021/22; switch says 65000 |

Most common cause: PE sub-interface dot1Q tag does not match the
switch VLAN ID. They MUST be identical (e.g. both 103 for CLIENT3).

### "CR sees the SVI but no default route"

OSPF on the switch is responsible for pushing `0.0.0.0/0` down.
Check on the switch:
```
show ip ospf 3                                  (process status)
show ip ospf 3 database external                (should list 0.0.0.0/0 as Type-5)
show running-config | section router ospf 3     (look for default-information originate always)
```

Fix: the `default-information originate always` line MUST be present.
Without `always`, the switch only originates a default if it sees one
in its own VRF table — which only happens after BGP delivers one.

### "DC unreachable from a CLIENT"

The community-RT scheme replaces the per-tenant import wall, so
verify the community RT is set both ends:

```
show vrf detail CLIENT3                         (on PE) — must show export 65000:1000
show vrf detail DC_PUB                          (on R3_E) — must show import 65000:1000
show bgp vpnv4 unicast all 203.0.113.0          (anywhere) — must show
                                                  RT 65000:99 + 65000:1000 attached
```

Common cause: missed `route-target export 65000:1000` on a freshly
added tenant. Without it, the tenant routes never carry the
TO_DC_PUB tag and DC_PUB's import sieve drops them.

For Slican (DC_CENTRAL) reachability — same idea but with `65000:2000`.
Remember: ONLY Site 2 tenants (CLIENT21-40 + SHARED_102) export
`65000:2000`. Site 1 tenants intentionally don't.

### "Two CLIENTs in different VRFs can ping each other (leak!)"

This should be impossible by design. If it happens:

```
show bgp vpnv4 unicast all                      (on the leaking PE)
show vrf detail CLIENT3                         (look at import/export RTs)
```

Verify CLIENT3 imports ONLY `65000:3` and `65000:99` (and `:100` on
Site 2). Any additional import is a leak.

Note: outlets WITHIN the same shared VRF (e.g. SHARED_101 outlets 4
and 7) CAN ping each other. That's the designed behavior — they
share one routing table on the switch.

### "ISIS adjacency keeps flapping"

```
show isis neighbors detail
show logging | include ISIS
```

Common causes:
- Interface MTU mismatch between adjacent routers.
- `isis network point-to-point` missing on a P2P link (defaults to
  broadcast and runs DR election that doesn't make sense on /30).
- `metric-style wide` set on one side but not the other.

### "SR-MPLS labels missing for a remote Loopback"

```
show segment-routing mpls connected-prefix-sid-map ipv4
show isis database verbose | include (SR|Capability)
show mpls forwarding-table 10.255.13.13
```

If the prefix-SID is missing for a Loopback: confirm the remote PE
has `isis prefix-sid index N` on its Loopback0. The index must be
unique across the IS-IS domain.

### "PE config copy fails over TFTP mid-paste"

The 3750-E configs are ~1290 lines. If `copy tftp: running-config`
truncates:
- Increase `parser command-history` and `terminal length`.
- Verify TFTP server isn't dropping the session (Tftpd64 default
  timeout is sane; Linux `tftpd-hpa` has `--timeout`).
- Try `copy tftp: flash:SW_AGR1.cfg` first, then
  `copy flash:SW_AGR1.cfg running-config`. The two-step bypasses
  network timing issues entirely.

---

## 5. Useful one-liners

```
! Count established BGP sessions on a PE
show bgp vpnv4 unicast all summary | count Established

! Show every VRF and its route count on a switch
show ip vrf
show ip route vrf * summary

! Dump the entire VPNv4 table grouped by RD (PE)
show bgp vpnv4 unicast all | begin Route Distinguisher

! What's in the OSPF VRF process N, one-liner
show ip ospf N | section Process

! VRF-aware ping from a PE (most common verification)
ping vrf CLIENT3 172.16.3.2
ping vrf CLIENT3 192.168.3.1
ping vrf CLIENT3 203.0.113.1
```

---

## 6. Reset-to-baseline procedure

If a switch / PE gets into a wedged state during testing:

1. **PE (ISR 4331)** — boot ROMmon, `confreg 0x2142` to ignore startup,
   `reload`. At prompt: `enable`, `copy usbflash0:R1_E.cfg running-config`,
   `write memory`, `config-register 0x2102`, `reload`.

2. **Catalyst (3750-E)** — at switch prompt: `write erase`, `reload`
   (do NOT save when asked). After fresh boot: `enable`, set a temp
   IP on `interface vlan 1`, `copy tftp: running-config`, `write memory`.

3. **Console-paste fallback** — if no flash/TFTP available: open the
   `.txt` file, `terminal length 0`, `configure terminal`, paste in
   blocks of ~200 lines with a pause between. Watch for `% Invalid
   input` lines and resolve before continuing.

---

## 7. What was NOT touched in the recent refactor (commit `773182a`)

For confidence when debugging post-refactor issues:

- `Core_Routers/R1_C.txt`, `R2_C.txt`, `R3_C.txt` — untouched since
  IS-IS/SR-MPLS migration (commit `6420cd8`).
- IS-IS NET strings on every PE — unchanged.
- SR-MPLS prefix-SID indexes — unchanged.
- iBGP VPNv4 RR peerings to 10.255.1.1 and 10.255.2.2 — unchanged.
- DC_PUB / DC_CENTRAL Loopback IPs and VLAN 500/510/600 SVIs — unchanged.
- Supermicro and Slican access ports on SW_AGR3 (Gi1/0/20-23) — unchanged.

What DID change: client VRF inventory, R1_E / R2_E sub-interface
forest, switch-side everything except management/SSH/STP, and the
DC import lists on R3_E (now community-RT only).
