# SONiC configuration

`config_db.json` is a lab-relevant subset of `sonic-cfggen -d --print-data`.
SONiC default tables (SNMP, default BGP neighbors, telemetry, auto-techsupport)
are omitted. `frr.conf` is the complete FRR configuration captured with:

```bash
docker exec bgp vtysh -c "show running-config"
```

SN21 has no management IPv4 in this lab; it is reached over the EVE console.
The FRR `password zebra` / `enable password zebra` defaults are redacted.
