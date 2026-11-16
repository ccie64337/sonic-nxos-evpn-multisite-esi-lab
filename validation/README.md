# Validation artifacts

- `traffic_filters.json` — the two native EVE-NG Pro traffic filters used in
  this lab. Add them through the lab UI (filter icon → Traffic Filters) or the
  Pro `/api/labs/<lab>/filter` API. They are not stored in the `.unl`.
- `dataplane_ep11_ep21.txt` — captured EP11 → EP21 ping (0% loss).

## Control plane

```text
# SPN11 / SPN21
show bgp l2vpn evpn summary
show bgp l2vpn evpn route-type 4

# BGW11 / BGW21
show nve vni
show nve peers
show bgp l2vpn evpn summary

# SN11 / SN21
show evpn es
show evpn es json
docker exec bgp vtysh -c "show bgp l2vpn evpn summary"
```

Expected:

- EVPN sessions Established in both sites.
- BGW VNI 10000 Up (`UnicastBGP`, MS-IR).
- Type-2/3 re-originated at the BGWs; Type-1 EAD exported.
- ESI `03:02:00:00:00:01:01:00:00:01` present on SN11/SN21.
- Type-1/2 preserve the ESI across the DCI; Type-4 is site-local.

## Data plane

```bash
# EP11
ping -c 5 10.100.100.13

# EP12
ping -c 5 10.100.100.11
ping -c 5 10.100.100.13
ping -c 5 10.100.100.1
```

Expected: 0% loss.

## Failover

```bash
# SN11
sonic-db-cli CONFIG_DB hset "PORTCHANNEL|PortChannel100" admin_status down
# verify EP12 reachability via SN21 (1 port, 0% loss)
sonic-db-cli CONFIG_DB hset "PORTCHANNEL|PortChannel100" admin_status up
```

Expected: 2 ports and 0% loss after restore.

## Traffic filters

Add `traffic_filters.json`, enable both filters, then:

```bash
ping -i 1 10.100.100.13
```

The ICMP filter highlights the host-facing links; the VXLAN filter highlights
the underlay.
