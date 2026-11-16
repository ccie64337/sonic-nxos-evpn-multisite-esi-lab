# Validation

## EVPN control plane

On SPN11/SPN21:

```text
show bgp l2vpn evpn summary
show bgp l2vpn evpn route-type 4
```

Expected: iBGP sessions to the local leaf, SONiC leaf, and BGW are Established;
the BGW's EVPN DCI session is Established.

On BGW11/BGW21:

```text
show nve vni
show nve peers
show bgp l2vpn evpn summary
```

Expected: VNI 10000 is `Up` with `UnicastBGP` and MS-IR; the remote BGW VIP is
an NVE peer; Type-2/3 are re-originated and Type-1 EAD is exported.

On SN11/SN21:

```text
show evpn es
show evpn es json
docker exec bgp vtysh -c "show bgp l2vpn evpn summary"
```

Expected: the ESI is present on both leaves; one leaf is DF and the other is
non-DF for the site-local Type-4 election.

Example:

```text
# SN11
show evpn es
ESI                            Type ES-IF                 VTEPs
03:02:00:00:00:01:01:00:00:01  LR   PortChannel100        2.2.2.21

# SN21
03:02:00:00:00:01:01:00:00:01  LRN  PortChannel100        1.1.1.21
```

`show evpn es json` shows `["local","remote"]` on SN11 and
`["local","remote","nonDF"]` on SN21 in the tested state.

## Type-4 scope

Type-4 is site-local. On SPN11:

```text
show bgp l2vpn evpn route-type 4
*>  [4]:[0302.0000.0001.0100.0001]:[32]:[1.1.1.21]  ET:8 ES-Import-Rt:...
```

On BGW11's NVE, the Type-4 is the BGW's own NVE:

```text
[4]:[0300.0000.0000.0100.0309]:[32]:[1.1.1.10]
```

LF11 has no ESI/Type-4 (NX-OS leaf is underlay/VTEP only). A remote site's
`show evpn es` shows only its local ES; this is expected Cisco multi-site
behavior.

## Type-1/2 ESI across the DCI

Type-2 for the host MAC is present on both leaves with the ESI:

```text
[2]:[0]:[48]:[de:c9:5f:2d:97:e7]              ESI:03:02:...:01
[2]:[0]:[48]:[de:c9:5f:2d:97:e7]:[32]:[10.100.100.12]  ESI:03:02:...:01
```

The BGWs preserve the ESI on Type-1/2 via
`PRESERVE_ESI_FABRIC`/`PRESERVE_ESI_DCI` (`set esi unchanged`).

## Data plane

From EP11:

```bash
ping -c 5 10.100.100.13     # EP11 -> EP21
```

From EP12:

```bash
ping -c 5 10.100.100.11     # EP12 -> EP11
ping -c 5 10.100.100.13     # EP12 -> EP21
ping -c 5 10.100.100.1      # EP12 -> anycast SVI
```

Expected: 0% loss. The tested lab shows ~16–19 ms RTT on the EP11↔EP21 path.

EP12 bond:

```text
Aggregator ID: 1
Number of ports: 2
Bonding Mode: IEEE 802.3ad Dynamic link aggregation
```

## Failover

On SN11:

```bash
sonic-db-cli CONFIG_DB hset "PORTCHANNEL|PortChannel100" admin_status down
```

Expected: SN11 ES becomes `LRN`, SN21 becomes `L` (DF), EP12 remains reachable
through SN21 with 1 port and 0% loss. Restore with `admin_status up`; EP12
returns to 2 ports and 0% loss.

## Native EVE Pro traffic filters

The lab includes:

- `EP11 <-> EP21 ICMP` — `icmp and host 10.100.100.11 and host 10.100.100.13`
- `VXLAN underlay` — `udp port 4789`

Open the lab, click the filter icon → Traffic Filters, enable both, and run:

```bash
ping -i 1 10.100.100.13
```

Matching links highlight in the filter color; the Traffic Graph/Details panel
shows live matches. The ICMP filter shows the host-facing links; the VXLAN
filter shows the encapsulated underlay path.

## Sanity commands

```text
# NX-OS
show ip route
show bgp ipv4 unicast summary
show nve peers
show nve vni

# SONiC
show vlan
show vxlan
show evpn es
ip route
```

## Known expected behavior

- Remote `show evpn es` only shows the local ES (Type-4 is site-local).
- LF11/LF21 show the ESI on Type-1/2 but have no local ESI/Type-4.
- The BGW NVE Type-4 is its own NVE, not the SONiC leaf Type-4.
- `dci-advertise-pip` remains disabled.
