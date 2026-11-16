# EVE Canvas

The EVE-NG canvas uses native EVE features only:

- **Text objects** for the SITE 1 box, SITE 2 box, DCI box, title, address-plan
  panel, legend, flow, validation, and shared-ESI panel.
- **Interface link labels** for the underlay/DCI/host/ESI link IPs.

The SVG/PNG in `topology/` is a standalone documentation diagram; EVE does not
load it.

## Site boxes

| Box | Position | Size | Title |
|---|---|---|---|
| SITE 1 | `153,120` | 660×450 | `🏢 SITE 1 — Site ID: 1 · AS 65001` |
| SITE 2 | `1308,120` | 660×450 | `🏢 SITE 2 — Site ID: 2 · AS 65002` |
| DCI | `829,119` | 450×115 | `☁️ DCI — eBGP 10.1.4.0/31` |

The site boxes list the devices, VLAN/VNI/SVI, hosts, and underlay links. The
DCI box lists the underlay and EVPN loopback peering.

## Node layout

| Site 1 | Position | Site 2 | Position |
|---|---|---|---|
| SPN11 | `400,260` | SPN21 | `1540,260` |
| LF11 | `240,440` | LF21 | `1540,440` |
| SN11 | `440,440` | SN21 | `1740,440` |
| BGW11 | `680,440` | BGW21 | `1340,440` |
| EP11 | `240,620` | EP21 | `1540,620` |
| EP12 | `900,620` | | |

BGW11/BGW21 are on the DCI-facing edges, so the DCI link stays clear of the
site 2 leaves.

## Link labels

EVE stores link labels on the interface:

```xml
<interface ... label="10.1.1.0/31" labelpos="0.5" />
```

EVE's topology API resolves the label from one endpoint, so the repo sets the
same label on both endpoints. Labels in this lab:

- `10.1.1.0/31`, `10.1.2.0/31`, `10.1.3.0/31`
- `DCI 10.1.4.0/31`
- `10.2.1.0/31`, `10.2.2.0/31`, `10.2.3.0/31`
- `VLAN 100 / 10.100.100.0/24` (LF11/LF21 host links)
- `ESI LAG / VLAN 100` (SN11/SN21 to EP12)

After changing the `.unl`, hard-refresh the EVE canvas (`Ctrl+F5`).

## Sanitization

The repository `.unl` is a sanitized copy: the lab-info text object uses
`[NXOS_PASSWORD]`, `[SONIC_PASSWORD]`, and `[LINUX_PASSWORD]` placeholders.
The live lab keeps its own credentials.

## Documentation canvas

`SONIC_NXOS_MULTISITE_ESI_CANVAS.svg` and `topology.png` show the same layout
with site boxes, site IDs, link labels, and the address plan. They are useful
for README/diagram export; they are not part of the EVE topology.

## Animated draw.io

`SONIC_ESI_MULTISITE_ANIMATED.drawio` is a draw.io/diagrams.net source with the
same topology. The inter-site path and the EP12 ESI LAG use
`flowAnimation=1`; open it in draw.io and use View/Present to see the moving
dashes. The `.drawio` file is documentation only and is not loaded by EVE-NG.
