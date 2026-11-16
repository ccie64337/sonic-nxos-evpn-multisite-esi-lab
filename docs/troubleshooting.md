# Troubleshooting

## BGW VNI is Down

The multi-site VNI needs **both** commands under `member vni 10000`:

```text
multisite ingress-replication
ingress-replication protocol bgp
```

Without `ingress-replication protocol bgp`, the VNI stays Down even though the
BGP EVPN session is Established.

## `dci-advertise-pip` breaks cross-site traffic

Keep it disabled:

```text
no dci-advertise-pip
```

Enabling it removed the remote VIP NVE peer and broke cross-site BUM/ARP in
this design.

## Remote `show evpn es` shows only the local ES

This is expected. Type-4 is site-local and is not re-originated across the DCI.
The ESI is carried on Type-1/2 and preserved by the BGW route-maps
(`set esi unchanged`).

## LF11/LF21 have no ESI/Type-4

N9Kv 10.6.3.F does not support ESI/EVPN-MH. LF11/LF21 are underlay/VTEP
devices; the ESI is SONiC↔SONiC (SN11/SN21). LF11/LF21 do see the ESI on
Type-1/2 routes.

## SONiC ESI disappears after a BGP restart

The ESI and iBGP neighbor are applied with FRR/`vtysh`, not CONFIG_DB. They do
not survive a `bgp` restart. Add a systemd hook such as
`/usr/local/bin/sonic-evpn-af.sh` + `/etc/sonic/evpn-mh.conf` for persistence.

## SN21 SSH is unreachable

SN21 has no management IPv4 in this lab. Use the EVE console. SN11 SSH works
after `iptables -F INPUT`; the SONiC image's default ACL can block SSH.

## LF11/LF21 SSH is unreachable

Their `mgmt0` has no IP. Use the EVE console, or add management IPs if needed.

## EVE link labels are not visible

EVE stores link labels on the interface (`label` + `labelpos`), and its
topology API resolves the label from one endpoint. Set the same label on **both**
endpoints of the link. After changing the `.unl`, hard-refresh the EVE canvas
(`Ctrl+F5`).

## Traffic Filters unavailable

The native Pro feature needs:

- EVE-NG Pro 6.4.0-50 or later.
- `lab-traffic-monitor` running:
  `ps aux | grep lab-traffic-monitor`
- Apache WebSocket config:
  `grep traffic-ws /etc/apache2/sites-enabled/eveng-ssl.conf`

The lab UI uses `/traffic-ws`; live matches are WebSocket-only. If the daemon
or WebSocket is down, the UI shows “Traffic Filter unavailable”.

## Traffic filter shows only host links, not the underlay

VXLAN encapsulates the host packet. The inner IPs are visible on the host-facing
links; the underlay links carry outer VTEP IPs and UDP 4789. Use a separate
`udp port 4789` filter to highlight the underlay path.

## EP12 has unexpected 10.99.99.x addresses

The captured host state includes stale `10.99.99.10` addresses on eth0/eth1
from an earlier lab. The active lab address is `10.100.100.12/24` on `bond0`.
Remove the stale addresses if they interfere:

```bash
ip addr del 10.99.99.10/24 dev eth0
ip addr del 10.99.99.10/24 dev eth1
ip route del default via 10.99.99.1 dev eth0
ip route del default via 10.99.99.1 dev eth1
ip route add default via 10.100.100.1
```

## Management IP confusion

The live management IPs are SPN11 `10.0.0.21`, SN11 `10.0.0.22`, BGW11
`10.0.0.33`, SPN21 `10.0.0.24`, BGW21 `10.0.0.34`. Earlier notes listing
BGW11 `.23`/BGW21 `.26` were incorrect. LF11/LF21/SN21 are console-only.
