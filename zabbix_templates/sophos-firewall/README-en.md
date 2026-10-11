# Sophos SFOS by SNMP

Monitoring template for **Sophos Firewall (SFOS / XG / XGS)** over SNMP v2c, with IPsec tunnel discovery by real connection name, wireless access points, PoE and licenses.

* **Author:** Pablo Pires de Godoi Silva
* **Zabbix version:** 7.4
* **MIB:** SFOS-FIREWALL-MIB v22 — enterprise OID `1.3.6.1.4.1.2604.5`
* **Tested on:** SFOS 22.0.2 MR-2 (Build 546), XGS138 appliance
* **Linked template:** `Network Generic Device by SNMP` (ships with Zabbix)

---

## Description

This template covers the whole SFOS-FIREWALL-MIB tree: device info, resources, temperatures, services, licenses, HA, fans, power supplies, PoE, IPsec VPN and the Sophos wireless access points.

All items use **numeric OIDs**, so the vendor MIB does not need to be installed on the Zabbix server or proxy.

### Tunnel traffic is named after the VPN, with no manual mapping

Sophos names tunnel interfaces `xfrm<N>` and returns an empty `ifAlias` for every interface, so interface-based traffic items are normally unreadable (`Interface xfrm1(): Bits received`).

It turns out that **`N` is the `sfosIPSecVpnConnId`** of the matching IPsec connection:

```
sfosIPSecVpnConnName.1  = "BRANCH_A"   ->  xfrm1
sfosIPSecVpnConnName.3  = "BRANCH_B"   ->  xfrm3
sfosIPSecVpnConnName.10 = "HQ_BACKUP"  ->  xfrm10
```

The discovery rule reads the IF-MIB and the IPsec tunnel table in a single walk and joins them by the number inside the interface name — never by the row index, since the two tables are indexed independently. The result is `VPN BRANCH_A: Download`, with zero per-host configuration.

### Collection watchdog

If SNMP stops answering, tunnel status items keep their last value and dashboards stay green. The item `sophos.ipsec.tunnels.count` exists to break that silence: with no collection, its `nodata` trigger fires.

### Noise control

Services and licenses that an environment does not use are switched off through context macros, instead of disabling items one by one.

---

## Requirements

1. SNMP enabled on the firewall: **Administration > SNMP**, SNMPv2c agent active with a read community.
2. The Zabbix server or proxy allowed in the SNMP ACL and in the firewall rule (UDP/161).
3. The official `Network Generic Device by SNMP` template present (bundled with Zabbix 7.4).

The vendor MIB is only useful for human reading and is available at
<https://docs.sophos.com/nsg/sophos-firewall/MIB/SOPHOS-XG-MIB.zip>.

---

## Setup

1. Import the template file.
2. Create the host with an **SNMP interface** pointing at the management IP.
3. Link the `Sophos SFOS by SNMP` template.
4. Set `{$SNMP_COMMUNITY}` on the host (inherited from the linked generic template).

---

## Macros

### Discovery filters

| Macro | Default | Description |
|---|---|---|
| `{$SOPHOS.VPN.NAME.MATCHES}` | `.*` | IPsec tunnels to discover |
| `{$SOPHOS.VPN.NAME.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | IPsec tunnels to ignore |
| `{$SOPHOS.AP.NAME.MATCHES}` | `.*` | Access points to discover |
| `{$SOPHOS.AP.NAME.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | Access points to ignore |

### Thresholds

| Macro | Default | Description |
|---|---|---|
| `{$SOPHOS.CPU.UTIL.CRIT}` | `90` | CPU utilization (%) |
| `{$SOPHOS.MEMORY.UTIL.MAX}` | `90` | Memory utilization (%) |
| `{$SOPHOS.SWAP.UTIL.MAX}` | `80` | Swap utilization (%) |
| `{$SOPHOS.DISK.UTIL.MAX}` | `90` | Disk utilization (%) |
| `{$SOPHOS.TEMP.CPU.WARN}` | `75` | CPU temperature, warning (°C) |
| `{$SOPHOS.TEMP.CPU.CRIT}` | `85` | CPU temperature, critical (°C) |
| `{$SOPHOS.POE.USAGE.MAX}` | `90` | PoE port draw, % of the configured limit |
| `{$SOPHOS.LICENSE.EXPIRE.WARN}` | `30` | Days ahead to warn about license expiry |
| `{$SOPHOS.IPSEC.NODATA}` | `10m` | Time without IPsec MIB data before alerting |

### Context macros

```
{$SOPHOS.SERVICE.MONITOR}              = 1    alert on every service
{$SOPHOS.SERVICE.MONITOR:"pop3"}       = 0    except POP3
{$SOPHOS.LICENSE.MONITOR:"webserver"}  = 0    ignore the Web Server license
```

Off by default, as the most common unused features: services `pop3`, `imap4`, `smtp`, `ftp`, `as`, `ha`; licenses `mail` and `webserver`.

---

## What is collected

**75 items.** Device name, model, firmware, Webcat and IPS signature versions; CPU (core average, via HOST-RESOURCES-MIB), memory, swap and disk; CPU and NPU temperature; 21 services; 9 license modules with status, expiry date and **computed days remaining**; HA status and peer state; wireless totals; five master walk items.

**7 discovery rules.**

| Discovery | Items per entity | Triggers |
|---|---|---|
| IPsec tunnels | Activated, Connection status, Connection type, Active tunnels | tunnel down (High), partially active (Warning) |
| IPsec tunnel traffic | Download, Upload — named after the VPN | — |
| Wireless access points | Status, Connected clients, Model, IP, MAC | AP disconnected (Average) |
| SSID | Connected clients | — |
| PoE ports | Power state, Limit, Draw, Voltage, Current | draw over the limit (Warning) |
| Fans | Speed | — |
| Power supplies | Status | — |

VPN items carry a `vpn:` tag, wireless items carry `ap:` and `ssid:` tags, so they can be filtered per tunnel or per network in *Latest data*, in *Problems* and in Grafana.

**Static triggers.** Disk, memory, swap and CPU utilization; CPU temperature at two levels; firmware change (Info); 21 service-stopped; 9 license-expired; 2 for the IPsec collection watchdog.

---

## Implementation notes

Places where the MIB or Zabbix behaves counter-intuitively. Worth reading before changing anything.

**`Activated` and `Connection status` are different things.** `sfosIPSecVpnActivated` is the configuration on/off switch; `sfosIPSecVpnConnStatus` is the operational state. Both triggers require `Activated = 1` so that a deliberately disabled connection never raises an incident.

**`sfosIPSecVpnActiveTunnel` changes meaning with the connection type.** On route-based connections (`tunnel-interface`) it is always 0 and useless. On a **remote access** connection it is the number of active client tunnels — how many users are connected to that profile. On those connections `Connection status` stays inactive even with users online, because there is no permanent tunnel; `Active tunnels` is what reflects real activity.

**`sfosIPSecVpnConnType` returns 0, a value the MIB does not enumerate.** The MIB defines only `host-to-host(1)`, `site-to-site(2)` and `tunnel-interface(3)`, but remote access connections answer 0. The value map covers it as `remote-access`.

**Access point status is inverted.** `APStatusType` is `connected(0)` and `disconnected(1)`.

**There is deliberately no per-client wireless item.** Wireless clients associate and leave constantly; discovering them would create and destroy items endlessly and fill the database with orphans. The client table is consumed as aggregates only — total and per SSID.

**A `SNMP walk to JSON` field name must not be an LLD macro inside an item prototype.** Zabbix substitutes LLD macros across the whole step parameter, JavaScript code included. Using `{#WIFI.SSID}` as the field name would turn `rows[i]['{#WIFI.SSID}']` into `rows[i]['Corp-WiFi']`, the key would no longer exist and the item would return 0 forever — **with no error at all**, looking like nobody uses the network. The prototypes therefore use plain field names (`ssid`, `status`, `name`).

**The wireless client table repeats the SSID.** One row per client means two clients on the same network would produce a duplicate item key and break discovery. The rule's JavaScript step de-duplicates first.

**`sfosDeviceAPMacAddress` is returned as hex.** The item uses format flag `2` of the *SNMP walk value* step, which converts it to MAC notation.

**The access point table is indexed by name.** `{#SNMPINDEX}` comes through as the name in dotted decimal; the readable name arrives in `{#AP.NAME}`, read from column 1.

**The MIB `LAST-UPDATED` field is not reliable.** The published file is stamped December 2018 while containing recent objects such as the PoE table. When in doubt about what a given firmware exposes, trust an `snmpwalk` over the file.

---

## Virtual appliances

A virtual appliance has no fans, power supplies, PoE or temperature sensors. On those hosts `sophos.health.walk` and `sophos.poe.walk` go **Not supported**, which is expected rather than a failure, the related discoveries create nothing, and the temperature items stay empty. Disable those master items on virtual hosts to avoid the noise.

---

## Security note

The `sfosIPSecVpnConnDes` column (`.3` of the tunnel table) is **deliberately not collected**. On firewalls managed by Sophos Central this field has been observed holding the same 64-character random string across every connection, strongly resembling a pre-shared key. If that is the case in your environment, anyone holding the read community can read it, and storing it in Zabbix history would only widen the exposure.

Compare the connection *Description* field with the *Pre-shared key* under **VPN > IPsec connections** to check. Either way, restrict SNMP with a source ACL and use a dedicated read-only community.

---

## Known limitations

* **No SSL VPN data.** The MIB exposes neither SSL VPN sessions nor a user count; the only related object is `sfosSSLVpnService`, which reports whether the daemon runs. Use syslog for session tracking. If remote access runs over IPsec, the `Active tunnels` item of the remote access connection answers the question.
* **An SSID with no clients is not discovered.** SSIDs come from the client table, so a configured but empty network only appears once someone associates. Already-discovered SSIDs correctly report 0.
* **License expiry dates are text.** The days-remaining calculation depends on the string format the firmware returns; check the `days to expire` items after the first collection.
* **The tunnel table is indexed by `ConnId`.** Item keys use `{#SNMPINDEX}`. If the firewall ever reuses a `ConnId` for a different connection, that item's history would mix the two. Not observed in normal use.

---

## License

MIT, as the rest of this repository. Sophos, SFOS, XG and XGS are trademarks of Sophos Ltd.; this template is not affiliated with or endorsed by the vendor.
