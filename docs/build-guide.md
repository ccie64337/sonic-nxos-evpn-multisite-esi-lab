# Build Guide

The complete running configurations are in `configs/`. This guide summarizes
the intent and the non-obvious steps.

## Prerequisites

- EVE-NG 6.x (Pro for traffic filters).
- N9Kv `nxosv9k-10.6.3.F` for SPN11/SPN21, LF11/LF21, BGW11/BGW21.
- SONiC `sonicsw-202511` for SN11/SN21.
- Linux hosts for EP11/EP12/EP21.
- Management network `10.0.0.0/24`, gateway `10.0.0.254`.

## 1. NX-OS underlay (eBGP local-as)

Example SPN11:

```text
feature bgp
feature nv overlay
nv overlay evpn

interface Ethernet1/1
  no switchport
  ip address 10.1.1.0/31
interface Ethernet1/2
  no switchport
  ip address 10.1.2.0/31
interface Ethernet1/3
  no switchport
  ip address 10.1.3.0/31
interface loopback0
  ip address 1.1.1.1/32

router bgp 65001
  router-id 1.1.1.1
  address-family ipv4 unicast
    network 1.1.1.1/32
  neighbor 10.1.1.1
    remote-as 65101
    local-as 65100 no-prepend replace-as
  neighbor 10.1.2.1
    remote-as 65102
    local-as 65100 no-prepend replace-as
  neighbor 10.1.3.1
    remote-as 65103
    local-as 65100 no-prepend replace-as
```

Each device uses its own local-as from `docs/ip-plan.md`. Do not add static
routes; the loopbacks are learned through eBGP.

## 2. iBGP EVPN overlay

SPN11 is the site-1 route reflector:

```text
router bgp 65001
  neighbor 1.1.1.10 remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
      route-reflector-client
  neighbor 1.1.1.11 remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
      route-reflector-client
  neighbor 1.1.1.21 remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
      route-reflector-client
```

SPN21 mirrors this with AS 65002 and `2.2.2.x` loopbacks.

## 3. NX-OS leaf / VTEP

LF11 (LF21 mirrors with `2.2.2.x`):

```text
vlan 100
  vn-segment 10000
interface Vlan100
  no shutdown
  ip address 10.100.100.1/24
  fabric forwarding mode anycast-gateway
interface nve1
  no shutdown
  source-interface loopback0
  host-reachability protocol bgp
  member vni 10000
    ingress-replication protocol bgp
evpn
  vni 10000 l2
    rd auto
    route-target both auto
```

## 4. EVPN Multi-Site BGW

BGW11 (BGW21 mirrors with site 2 / AS 65002):

```text
evpn multisite border-gateway 1
interface loopback100
  ip address 1.1.1.100/32
interface nve1
  no shutdown
  source-interface loopback0
  host-reachability protocol bgp
  multisite border-gateway interface loopback100
  member vni 10000
    multisite ingress-replication
    ingress-replication protocol bgp
interface Ethernet1/1
  no switchport
  ip address 10.1.4.0/31
  evpn multisite dci-tracking
interface Ethernet1/7
  no switchport
  ip address 10.1.3.1/31
  evpn multisite fabric-tracking

route-map PRESERVE_ESI_FABRIC permit 10
  set esi unchanged
route-map PRESERVE_ESI_DCI permit 10
  set esi unchanged

router bgp 65001
  router-id 1.1.1.10
  rd dual id 1
  neighbor 1.1.1.1
    remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
      route-map PRESERVE_ESI_FABRIC out
  neighbor 2.2.2.10
    remote-as 65203
    local-as 65103 no-prepend replace-as
    update-source loopback0
    ebgp-multihop 2
    peer-type fabric-external
    address-family l2vpn evpn
      send-community extended
      route-map PRESERVE_ESI_DCI out
  neighbor 10.1.4.1
    remote-as 65203
    local-as 65103 no-prepend replace-as
    address-family ipv4 unicast
evpn
  vni 10000 l2
    rd auto
    route-target import auto
    route-target import 65000:10000
    route-target export auto
    route-target export 65000:10000
```

Do **not** enable `dci-advertise-pip` with this design.

## 5. SONiC leaf (SN11 example)

CONFIG_DB objects:

```bash
sonic-db-cli CONFIG_DB hset "VLAN|Vlan100" vlanid 100
sonic-db-cli CONFIG_DB hset "PORTCHANNEL|PortChannel100" admin_status up mtu 9100
sonic-db-cli CONFIG_DB hset "PORTCHANNEL_MEMBER|PortChannel100|Ethernet0" NULL NULL
sonic-db-cli CONFIG_DB hset "VLAN_MEMBER|Vlan100|PortChannel100" tagging_mode untagged
sonic-db-cli CONFIG_DB hset "VXLAN_TUNNEL|vtep" src_ip 1.1.1.21
sonic-db-cli CONFIG_DB hset "VXLAN_TUNNEL_MAP|vtep|Vlan100" vni 10000 vlan Vlan100
sonic-db-cli CONFIG_DB hset "VXLAN_EVPN_NVO|nvo" source_vtep vtep
sudo config interface ip add Vlan100 10.100.100.1/24
```

FRR:

```text
router bgp 65001
 bgp router-id 1.1.1.21
 neighbor 1.1.1.1 remote-as 65001
 neighbor 1.1.1.1 update-source Loopback0
 neighbor 10.1.2.0 remote-as 65100
 neighbor 10.1.2.0 local-as 65102 no-prepend replace-as
 address-family ipv4 unicast
  network 1.1.1.21/32
  network 10.100.100.0/24
  neighbor 10.1.2.0 activate
 address-family l2vpn evpn
  neighbor 1.1.1.1 activate
  advertise-all-vni
interface PortChannel100
 evpn mh es-id 1
 evpn mh es-sys-mac 02:00:00:00:01:01
router bgp 65001
 evpn mh startup-delay 5
```

Apply with `docker exec bgp vtysh`. SN21 mirrors with `2.2.2.21`,
`10.2.2.0`, and local-as `65202`.

## 6. Hosts

EP11 / EP21:

```bash
ip addr add 10.100.100.11/24 dev eth0
ip link set eth0 up
ip route add default via 10.100.100.1
```

EP12:

```bash
ip link add bond0 type bond mode 802.3ad miimon 100 lacp_rate fast xmit_hash_policy layer3+4
ip link set eth0 down
ip link set eth1 down
ip link set eth0 master bond0
ip link set eth1 master bond0
ip link set bond0 up
ip addr add 10.100.100.12/24 dev bond0
ip route add default via 10.100.100.1
```

## 7. EVE canvas

- SITE 1 / SITE 2 / DCI boxes are native EVE text objects.
- Link IPs are native interface `label` attributes; set the same label on both
  endpoints so EVE's topology API resolves it.
- The site boxes use `Site ID: 1` / `Site ID: 2` in their titles.
- The SVG/PNG in `topology/` is documentation, not loaded by EVE.

## 8. Optional: native EVE Pro traffic filters

Add the two filters from `validation/traffic_filters.json` via the lab UI
(filter icon → Traffic Filters) or the Pro API. See
[`traffic-filters.md`](traffic-filters.md).

## 9. Save

On NX-OS: `copy running-config startup-config`.
On SONiC: save CONFIG_DB and the FRR config; ESI/FRR is not persistent unless
you add a systemd hook.
