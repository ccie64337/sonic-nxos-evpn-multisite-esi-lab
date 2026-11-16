# IP / ASN Plan

## Management

| Device | Management IP | Access |
|---|---|---|
| SPN11 | `10.0.0.21/24` | SSH / console |
| SN11 | `10.0.0.22/24` | SSH / console |
| BGW11 | `10.0.0.33/24` | SSH / console |
| SPN21 | `10.0.0.24/24` | SSH / console |
| BGW21 | `10.0.0.34/24` | SSH / console |
| LF11 | — | console only (mgmt0 no IP) |
| LF21 | — | console only (mgmt0 no IP) |
| SN21 | — | console only (no IPv4) |

Management gateway: `10.0.0.254`.

## Loopbacks

| Device | Loopback0 | Notes |
|---|---|---|
| SPN11 | `1.1.1.1/32` | spine / RR |
| LF11 | `1.1.1.11/32` | NX-OS leaf / VTEP |
| SN11 | `1.1.1.21/32` | SONiC leaf / VTEP |
| SN11 | `10.1.0.1/32` | additional loopback advertised by FRR |
| BGW11 | `1.1.1.10/32` | multi-site BGW |
| BGW11 | `1.1.1.100/32` | multi-site VIP (`loopback100`) |
| SPN21 | `2.2.2.1/32` | spine / RR |
| LF21 | `2.2.2.11/32` | NX-OS leaf / VTEP |
| SN21 | `2.2.2.21/32` | SONiC leaf / VTEP |
| BGW21 | `2.2.2.10/32` | multi-site BGW |
| BGW21 | `2.2.2.100/32` | multi-site VIP (`loopback100`) |

## Underlay links

| Link | Subnet | Local / remote |
|---|---|---|
| SPN11 E1/1 ↔ LF11 E1/7 | `10.1.1.0/31` | SPN11 `.0` / LF11 `.1` |
| SPN11 E1/2 ↔ SN11 Ethernet32 | `10.1.2.0/31` | SPN11 `.0` / SN11 `.1` |
| SPN11 E1/3 ↔ BGW11 E1/7 | `10.1.3.0/31` | SPN11 `.0` / BGW11 `.1` |
| BGW11 E1/1 ↔ BGW21 E1/1 | `10.1.4.0/31` | BGW11 `.0` / BGW21 `.1` |
| SPN21 E1/1 ↔ LF21 E1/7 | `10.2.1.0/31` | SPN21 `.0` / LF21 `.1` |
| SPN21 E1/2 ↔ SN21 Ethernet32 | `10.2.2.0/31` | SPN21 `.0` / SN21 `.1` |
| SPN21 E1/3 ↔ BGW21 E1/7 | `10.2.3.0/31` | SPN21 `.0` / BGW21 `.1` |

The EVE node interface is named `Ethernet9`; the SONiC OS names the same link
`Ethernet32`.

## BGP / ASN

| Device | Global AS (iBGP EVPN) | Underlay local-as |
|---|---|---|
| SPN11 | 65001 | 65100 |
| LF11 | 65001 | 65101 |
| SN11 | 65001 | 65102 |
| BGW11 | 65001 | 65103 |
| SPN21 | 65002 | 65200 |
| LF21 | 65002 | 65201 |
| SN21 | 65002 | 65202 |
| BGW21 | 65002 | 65203 |

DCI:

- Underlay: `10.1.4.0/31` (BGW11 `.0` ↔ BGW21 `.1`).
- EVPN: eBGP over loopbacks `1.1.1.10` ↔ `2.2.2.10`, `ebgp-multihop 2`,
  `peer-type fabric-external`.

## VXLAN / EVPN

| Item | Value |
|---|---|
| VLAN | 100 |
| VNI | 10000 |
| Anycast SVI | `10.100.100.1/24` |
| Anycast gateway MAC | `22:63:39:58:ff:0a` |
| ESI | `03:02:00:00:00:01:01:00:00:01` |
| ESI ID | 1 |
| ESI system MAC | `02:00:00:00:01:01` |
| Route target | `65000:10000` |
| Site IDs | 1 / 2 |

## Hosts

| Host | IP | Attachment |
|---|---|---|
| EP11 | `10.100.100.11/24` | LF11 E1/1 (VLAN 100) |
| EP12 | `10.100.100.12/24` | `bond0` LACP to SN11/SN21 (ESI) |
| EP21 | `10.100.100.13/24` | LF21 E1/1 (VLAN 100) |

Default gateway: `10.100.100.1` (anycast SVI).

EP12's captured host state also shows stale `10.99.99.10` addresses on eth0/eth1
from an earlier lab; the active lab address is on `bond0`.
