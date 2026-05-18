# TODO — Multi-Site WAN Lab

Deferred quality enhancements layered on top of the T1 routing baseline. None of these touch IS-IS, SR-MPLS or BGP RR topology — they add QoS / MTU / MSS hygiene to make VoIP production-grade under WAN contention.

Implement after the pending design change lands.

---

## P1 — VoIP Quality of Service

### Switches (Catalyst 3750-E)
Apply to `SW_AGR1`, `SW_AGR2`, `SW_AGR3`:

- `mls qos` globally — required to enable any QoS on the 3750-E (off by default; without it, all DSCP is rewritten to 0)
- On `Gi1/0/1` (PE-facing trunk): `auto qos voip trust`
- On hardphone access ports: `auto qos voip cisco-phone` + `mls qos trust device cisco-phone`
- Verify EF -> priority queue: `show mls qos interface Gi1/0/1 queueing`

### PE Routers (ISR 4331)
Apply to `R1_E`, `R2_E`, `R3_E`:

- Input policy on `Gi0/0/0.X` sub-interfaces: trust DSCP from CE
- Output policy on `Gi0/0/2` (WAN uplink toward core):
  - EF (DSCP 46) -> `priority percent 33`
  - CS3 (DSCP 24) -> `bandwidth percent 5` for SIP signalling
  - Default class -> `fair-queue`

### P Routers (ISR 4331)
Apply to `R1_C`, `R2_C`, `R3_C`:

- Mirror the LLQ output policy on all core P2P links
- MPLS EXP <-> DSCP uniform-mode propagation is the IOS-XE default — verify, no extra config expected

---

## P2 — MTU & MSS Hygiene

Compensates for the 8-byte MPLS overhead (SR transport SID + VPN label).

- All P2P core links (`R*C` Gi0/0/0-2, `R*E` Gi0/0/2): `mtu 1600`
- 3750-E switches: `system mtu 1546` + reload (system MTU change requires reboot)
- PE sub-interfaces (`R*E Gi0/0/0.X`) per VRF: `ip tcp adjust-mss 1360`
- Validate end-to-end: `ping <remote-PE-loopback> source <local-PE-loopback> df-bit size 1500`

---

## P3 — Voice VLAN Segmentation (Optional, hardphones only)

Skip entirely if VoIP endpoints are softphones on PCs.

- Per client site: dedicated voice VLAN (proposed range 401–405) inside the corresponding CLIENT VRF
- SVI as voice gateway in CLIENT VRF on each `SW_AGRx`
- Access ports: `switchport access vlan 30X` + `switchport voice vlan 40X`
- `mls qos trust device cisco-phone` for LLDP-MED conditional trust
- Extend the per-client OSPF VRF process with the new voice subnet so the default route reaches phones

---

## Out of Scope (for now)

- Multicast (paging, MoH) — Slican typically unicast-only, no PIM needed
- SRTP / TLS-SIP — handled at the PBX endpoint, transparent to the network
- PE redundancy / dual-homing — physical cable constraint, accepted as-is
- WAN encryption (MACsec, IPsec over core) — out of scope for T1 lab

---

## Reminder

Inter-branch VoIP hairpins through Slican by design (RT policy blocks direct CLIENT <-> CLIENT). Size the Slican uplink for `2 × concurrent_calls` of RTP bandwidth (e.g. G.711 ~87 kbps × 2 = ~175 kbps per active inter-site call).
