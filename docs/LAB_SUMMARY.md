# Lab Summary

**One-line description:** Two-site SONiC/NX-OS EVPN multi-site ESI lab in EVE-NG with an eBGP underlay, iBGP EVPN overlay, Cisco EVPN Multi-Site BGWs, and a dual-homed SONiC ESI host.

## Overview

The lab models two EVPN sites joined by a Cisco EVPN Multi-Site DCI. Each site
has a Cisco Nexus 9000v spine acting as the BGP route reflector, an N9Kv leaf
and VTEP, an N9Kv EVPN Multi-Site border gateway, and a SONiC leaf providing
RFC 7432 ESI. EP12 is LACP-bonded across the two SONiC leaves and uses the
anycast gateway.

The underlay is eBGP with a unique `local-as` per device and
`no-prepend replace-as`. The overlay is iBGP EVPN with the spines as route
reflectors. The BGWs peer eBGP over loopbacks across the DCI with
`peer-type fabric-external` and preserve the ESI on Type-1/2.

## Topology

| Site | Spine | NX-OS leaf | Border GW | SONiC leaf | Hosts |
|---|---|---|---|---|---|
| 1 (Site ID 1) | SPN11 | LF11 | BGW11 | SN11 | EP11, EP12 (ESI bond) |
| 2 (Site ID 2) | SPN21 | LF21 | BGW21 | SN21 | EP21, EP12 (ESI bond) |

## Design

- **Underlay:** eBGP on every fabric link; global AS 65001/65002 for the
  overlay; underlay local-as SPN11 65100, LF11 65101, SN11 65102,
  BGW11 65103; SPN21 65200, LF21 65201, SN21 65202, BGW21 65203.
- **Overlay:** iBGP EVPN; SPN11/SPN21 are route reflectors.
- **DCI:** BGW11 ↔ BGW21; IPv4 underlay `10.1.4.0/31`; EVPN eBGP over
  loopbacks `1.1.1.10` ↔ `2.2.2.10`, `ebgp-multihop 2`,
  `peer-type fabric-external`.
- **VXLAN:** VLAN 100, VNI 10000, anycast SVI `10.100.100.1/24`.
- **ESI:** `03:02:00:00:00:01:01:00:00:01` on `PortChannel100` on SN11/SN21;
  EP12 `bond0` 802.3ad (LACP).
- **Multi-site:** site IDs 1/2, `rd dual id 1/2`, MS-IR,
  `PRESERVE_ESI_FABRIC`/`PRESERVE_ESI_DCI` with `set esi unchanged`;
  `dci-advertise-pip` disabled.
- **Type-4:** site-local. Type-1/2 ESI is preserved across the DCI.

## Validation

| Check | Result |
|---|---|
| EP11 ↔ EP21 | 0% loss |
| EP12 ↔ EP11 / EP21 | 0% loss |
| EP12 → anycast SVI | 0% loss |
| BGW VNI 10000 | Up (`UnicastBGP`, MS-IR) |
| Type-1/2 ESI across DCI | preserved |
| Type-4 | site-local (expected) |

EP12 failover was tested by disabling `PortChannel100` on SN11: EP12 stayed
reachable through SN21 (1 port, 0% loss), then returned to 2 ports after
restore.

## Current state

- EVE-NG **7.4.0-2-PRO** on Ubuntu 24.04.
- Lab nodes are stopped after the EVE-NG upgrade.
- NX-OS startup configs are saved.
- SONiC FRR/ESI was runtime-only; reapply from
  `configs/sonic/SN11/frr.conf` and `configs/sonic/SN21/frr.conf` after boot.

## Artifacts

- Lab file: `topology/SONIC_ESI_MULTISITE.unl`
- Topology preview: `topology/topology.png`
- Animated diagram: `docs/SONIC_ESI_MULTISITE_ANIMATED.svg`
- Configs: `configs/nxos/`, `configs/sonic/`, `configs/hosts/`
- Details: `docs/architecture.md`, `docs/ip-plan.md`,
  `docs/build-guide.md`, `docs/validation.md`
