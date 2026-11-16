# Native EVE-NG Traffic Filters (Pro)

The animated/live link highlighting is a **native EVE-NG Pro feature**, not a
plugin or custom SVG overlay. It was introduced in EVE-NG Pro 6.4.0-50
(December 2025) as “Lab traffic detection & filtering”. This lab runs on
EVE-NG **6.5.0-27-PRO**.

## What it does

A filter is a BPF (tcpdump/pcap) expression with a color and timeout. The
`lab-traffic-monitor` daemon watches lab traffic and the lab UI receives
matches over `/traffic-ws`. Matching links get the `traffic-active` class and
are highlighted/animated; a Traffic Graph/Details panel shows live matches.

It animates **live matching traffic**. It is not a looping decorative
animation; if no packets match, nothing animates.

## Add a filter in the UI

1. Open the lab canvas.
2. Click the filter icon in the toolbar → **Traffic Filters**.
3. Click **NEW FILTER** (or **Add First Filter**).
4. Fill in:
   - **Title** — e.g. `EP11 <-> EP21 ICMP`
   - **Filter expression** — BPF syntax, e.g.
     `icmp and host 10.100.100.11 and host 10.100.100.13`
   - **Color** — e.g. `#00e5ff`
   - **Timeout** — 50–60000 ms (5000 is a good default)
   - **Enabled** — on
5. Start traffic, e.g. from EP11:

   ```bash
   ping -i 1 10.100.100.13
   ```

The matching links highlight in the filter color. Use **All Enabled/All
Disabled** to toggle filters; per-filter visibility is stored in browser
localStorage (`traffic_filter_visibility`).

## Demo filters in this lab

`validation/traffic_filters.json` contains:

| Title | BPF expression | Color |
|---|---|---|
| `EP11 <-> EP21 ICMP` | `icmp and host 10.100.100.11 and host 10.100.100.13` | `#00e5ff` |
| `VXLAN underlay` | `udp port 4789` | `#ffb84d` |

The first shows the host-facing ICMP links; the second shows the VXLAN
underlay. Because VXLAN encapsulates the host packet, the underlay links carry
outer VTEP IPs and UDP 4789, not the inner host IPs.

## Pro API

The UI uses these endpoints (authentication required):

```text
GET    /api/labs/<lab>/filter
POST   /api/labs/<lab>/filter
POST   /api/labs/<lab>/filter/activate
PUT    /api/labs/<lab>/filter/<uuid>
DELETE /api/labs/<lab>/filter/<uuid>
```

Example create payload:

```json
{
  "title": "EP11 <-> EP21 ICMP",
  "filter_expr": "icmp and host 10.100.100.11 and host 10.100.100.13",
  "enabled": 1,
  "color": "#00e5ff",
  "timeout": 5000
}
```

## Troubleshooting

- The feature is **Pro-only**; Community edition has no native link animation.
- Check the daemon: `ps aux | grep lab-traffic-monitor`.
- Check Apache: `grep traffic-ws /etc/apache2/sites-enabled/eveng-ssl.conf`.
- The UI receives matches over `/traffic-ws`; if the WebSocket is down, the UI
  shows “Traffic Filter unavailable”.
- If you want a permanent demo without real traffic, run a continuous
  `ping -i 1` or `iperf3` in the background; EVE only animates matching packets.
