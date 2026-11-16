# Configurations

These are sanitized copies of the live lab configurations.

- `nxos/` — full `show running-config` from SPN11, SPN21, LF11, LF21,
  BGW11, and BGW21. `username` password hashes and SNMP credential lines
  are removed.
- `sonic/SN11/`, `sonic/SN21/` — SONiC FRR configuration (BGP/EVPN/ESI)
  and a lab-relevant subset of `sonic-cfggen -d --print-data`.
- `hosts/` — EP11/EP12/EP21 Linux host state (`ip addr`, routes, bond).

The configs are reference material; replace management IPs, credentials,
and image paths for your own EVE-NG environment.
