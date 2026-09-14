# Airtel 5G NSA Setup for ZX7981PG

This guide records the Airtel profile used on the ZX7981PG. Airtel NSA requires an LTE anchor plus the NR n78 secondary carrier; seeing LTE or 4G+ alone does not prove that n78 is attached.

> [!IMPORTANT]
> NSA attachment is controlled by tower availability, LTE anchor eligibility, SIM provisioning, load and RF conditions. These commands enable capability; they cannot force a tower to provide NSA.

## Working profile

| Setting | Tested value |
|---|---|
| APN | `airtelgprs.com` |
| Router IPv6 selector | `1` (IPv4 + IPv6) |
| Dial process | `quectel-CM -4 -6 -s airtelgprs.com` |
| RAT preference | `AUTO` |
| 5G disable mode | `0` |
| NSA NR band | n78 |

## 1. Save and apply the Airtel profile

```sh
uci set lte.main.apn='airtelgprs.com'
uci set lte.main.ipv6='1'
uci commit lte
/etc/init.d/lte restart
sleep 30
```

Confirm the dial process:

```sh
ps w | grep '[q]uectel-CM'
```

Expected command:

```text
quectel-CM -4 -6 -s airtelgprs.com
```

## 2. Enable automatic LTE/NR selection

```sh
ubus -t 8 call lteat send '{"cmd":"AT+QNWPREFCFG=\"mode_pref\",AUTO"}'
ubus -t 8 call lteat send '{"cmd":"AT+QNWPREFCFG=\"nr5g_disable_mode\",0"}'
ubus -t 8 call lteat send '{"cmd":"AT+QNWPREFCFG=\"nsa_nr5g_band\",78"}'
```

Remove a previous LTE cell lock if one was deliberately configured:

```sh
ubus -t 8 call lteat send '{"cmd":"AT+QNWLOCK=\"common/4g\",0"}'
```

Do not run the unlock command speculatively on a different modem model or firmware.

## 3. Verify the data session

```sh
ubus -t 8 call lteat send '{"cmd":"AT+CGACT?"}'
ubus -t 8 call lteat send '{"cmd":"AT+CGPADDR=1"}'
ip link show dev wwan0_1
ip route show default
ping -c 4 1.1.1.1
```

## 4. Verify NSA, not only LTE carrier aggregation

Run the radio checks while actively downloading data, because an NSA secondary carrier may be released when idle:

```sh
ubus -t 10 call lteat send '{"cmd":"AT+QENG=\"servingcell\""}'
ubus -t 8 call lteat send '{"cmd":"AT+QNWINFO"}'
ubus -t 8 call lteat send '{"cmd":"AT+QENDC"}'
ubus -t 10 call lteat send '{"cmd":"AT+QCAINFO"}'
ubus -t 8 call lteat send '{"cmd":"AT+QRSRP"}'
```

Interpretation:

- `NR5G-NSA` in `QENG` confirms the NR secondary carrier.
- `QCAINFO` entries containing only LTE PCC/SCC carriers confirm 4G carrier aggregation, not 5G NSA.
- `QNWINFO: "FDD LTE"` alone indicates the current serving/anchor RAT, not an active n78 attachment.
- `nr5g_disable_mode,0` means NSA capability is enabled; it does not mean NSA is currently attached.

If the same Airtel SIM is slow and shows NSA disconnected in a known-good phone at the same location and time, the likely cause is tower/network availability rather than the router profile.

