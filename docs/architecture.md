# Architecture

## Overview

The lab is two EVPN sites connected by a Cisco EVPN Multi-Site DCI:

- **Site 1** (Site ID 1, AS 65001): SPN11, LF11, BGW11, SN11.
- **Site 2** (Site ID 2, AS 65002): SPN21, LF21, BGW21, SN21.
- **DCI**: BGW11 ↔ BGW21.

Each site has one N9Kv spine acting as the iBGP EVPN route reflector, one N9Kv
leaf, one N9Kv multi-site border gateway, and one SONiC leaf with ESI. The
spine/leaf/BGW provide the routed underlay and the multi-site control plane;
the SONiC leaves provide the RFC 7432 ESI access layer.

## Underlay — eBGP with per-device local-as

Every fabric link is eBGP. Each device keeps its global BGP AS for the iBGP
EVPN overlay and uses a unique `local-as` for the underlay session, with
`no-prepend replace-as` so the local-as is used as the peer AS without
prepending.

| Device | Global AS | Underlay local-as |
|---|---|---|
| SPN11 | 65001 | 65100 |
| LF11 | 65001 | 65101 |
| SN11 | 65001 | 65102 |
| BGW11 | 65001 | 65103 |
| SPN21 | 65002 | 65200 |
| LF21 | 65002 | 65201 |
| SN21 | 65002 | 65202 |
| BGW21 | 65002 | 65203 |

Underlay links:

| Link | Subnet |
|---|---|
| SPN11 E1/1 ↔ LF11 E1/7 | `10.1.1.0/31` |
| SPN11 E1/2 ↔ SN11 Ethernet32 | `10.1.2.0/31` |
| SPN11 E1/3 ↔ BGW11 E1/7 | `10.1.3.0/31` |
| BGW11 E1/1 ↔ BGW21 E1/1 | `10.1.4.0/31` |
| SPN21 E1/1 ↔ LF21 E1/7 | `10.2.1.0/31` |
| SPN21 E1/2 ↔ SN21 Ethernet32 | `10.2.2.0/31` |
| SPN21 E1/3 ↔ BGW21 E1/7 | `10.2.3.0/31` |

The EVE link names on the SONiC nodes are `Ethernet9`; the SONiC OS exposes the
same link as `Ethernet32`. Static routes are not used; loopbacks are reachable
through eBGP.

## Overlay — iBGP EVPN

- Site 1: iBGP AS 65001; SPN11 is the route reflector.
- Site 2: iBGP AS 65002; SPN21 is the route reflector.
- SPN11 clients: LF11 `1.1.1.11`, BGW11 `1.1.1.10`, SN11 `1.1.1.21`.
- SPN21 clients: LF21 `2.2.2.11`, BGW21 `2.2.2.10`, SN21 `2.2.2.21`.
- `address-family l2vpn evpn`, `send-community extended`.
- The SONiC leaves use FRR BGP with `advertise-all-vni`.

## VXLAN and anycast gateway

- VLAN 100, VNI 10000.
- Anycast SVI `10.100.100.1/24` on the leaves; anycast gateway MAC
  `22:63:39:58:ff:0a`.
- VTEPs: SN11 `1.1.1.21`, SN21 `2.2.2.21`, LF11 `1.1.1.11`, LF21 `2.2.2.11`,
  BGW11 `1.1.1.10`, BGW21 `2.2.2.10`.
- EP11 `10.100.100.11`, EP12 `10.100.100.12`, EP21 `10.100.100.13`.

## ESI

- ESI `03:02:00:00:00:01:01:00:00:01`.
- `PortChannel100` on SN11 and SN21, member `Ethernet0`.
- `evpn mh es-id 1`, `evpn mh es-sys-mac 02:00:00:00:01:01`.
- EP12 uses `bond0` 802.3ad (LACP), one member to each SONiC leaf.
- The ESI is applied with FRR/`vtysh`; it is not a CONFIG_DB object.

Type-4 DF election is **site-local**. Type-1/2 routes carry the ESI and the
BGWs preserve it across the DCI, so the remote site sees the ESI for the
dual-homed host. A remote `show evpn es` shows only the local ES; this is
expected.

## EVPN Multi-Site BGW

- BGW11: `evpn multisite border-gateway 1`; BGW21: `2`.
- Multi-site VIP: `1.1.1.100` / `2.2.2.100` on `loopback100`.
- `interface nve1`:
  - `multisite border-gateway interface loopback100`
  - `member vni 10000`
  - `multisite ingress-replication`
  - `ingress-replication protocol bgp`
- DCI interface: `evpn multisite dci-tracking`.
- Fabric interface: `evpn multisite fabric-tracking`.
- `router bgp` has `rd dual id 1` / `rd dual id 2`.
- `evpn vni 10000 l2` uses `rd auto` and route-targets `auto` plus
  `65000:10000`.
- Route-maps `PRESERVE_ESI_FABRIC` and `PRESERVE_ESI_DCI` use
  `set esi unchanged`; Type-1 is permitted so the ESI EAD route crosses the DCI.
- The BGW EVPN DCI peering is eBGP over loopbacks
  (`1.1.1.10` ↔ `2.2.2.10`, `ebgp-multihop 2`,
  `peer-type fabric-external`). The `10.1.4.0/31` link is the IPv4 underlay
  that makes those loopbacks reachable.
- **`dci-advertise-pip` is disabled.** Enabling it removed the remote VIP NVE
  peer and broke cross-site BUM/ARP.

## Traffic path

EP11 → EP21:

1. EP11 sends to the anycast SVI on LF11.
2. LF11 VXLAN-encapsulates to LF21 (remote VTEP) over the underlay.
3. The underlay path is LF11 → SPN11 → BGW11 → DCI → BGW21 → SPN21 → LF21.
4. LF21 decapsulates and forwards to EP21.

EP12 is dual-homed to SN11/SN21. Local traffic uses the local SONiC leaf;
during a leaf/port failure, traffic uses the other member of the ESI LAG.

## EVE-NG traffic filters

EVE-NG Pro 6.4.0-50+ adds native live Traffic Filters. The lab includes two
filters (ICMP host flow and VXLAN underlay). See
[`traffic-filters.md`](traffic-filters.md).

## Design caveats

- N9Kv 10.6.3.F has no ESI/EVPN-MH; do not expect NX-OS↔SONiC ESI.
- LF11/LF21 are underlay/VTEP devices, not ESI peers.
- SONiC ESI/FRR state is not persistent across a `bgp` restart.
- SN21 and LF11/LF21 are console-only in this lab (no management IPv4).
- The BGW VNI requires both ingress-replication commands or it stays Down.
