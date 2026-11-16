# Topology

- `SONIC_ESI_MULTISITE.unl` — EVE-NG lab file. Import it under
  `/Users/<user>/SONIC_ESI_MULTISITE.unl`. Credentials in the lab-info text
  object are replaced with placeholders in this repository copy.
- `SONIC_NXOS_MULTISITE_ESI_CANVAS.svg` — standalone documentation canvas.
  It is not loaded by EVE-NG.
- `topology.png` — PNG render of the documentation canvas for the README.

The EVE canvas uses native text objects for the SITE 1/SITE 2/DCI boxes and
native interface `label` attributes for the link IPs. The traffic filters are
stored by EVE-NG Pro, not in the `.unl`.
