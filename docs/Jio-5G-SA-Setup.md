# Jio 5G SA Setup for ZX7981PG

This guide records the configuration successfully used with a Jio SIM on the ZX7981PG. The observed network was **NR5G-SA**, band **n78**, with Jio's IPv6-only PDP service and automatic 464XLAT for IPv4 destinations.

> [!IMPORTANT]
> Back up `/etc/config/lte` and `/etc/config/network` before changing anything. Commands and interface names in this guide are specific to the tested firmware. Do not publish screenshots containing IMEI, IMSI, ICCID, phone number, MAC address or public IP.

## Working profile

| Setting | Tested value |
|---|---|
| Operator | Jio (`405/871`) |
| APN | `jionet` |
| Router IPv6 selector | `2` (IPv6 only) |
| Dial process | `quectel-CM -6 -s jionet` |
| RAT preference | `AUTO` |
| 5G disable mode | `0` |
| SA band | n78 |
| IPv4 compatibility | Automatic 464XLAT |

## 1. Back up the current configuration

```sh
cp /etc/config/lte /root/lte.backup
cp /etc/config/network /root/network.backup
```

## 2. Save the Jio APN and IPv6-only mode

```sh
uci set lte.main.apn='jionet'
uci set lte.main.ipv6='2'
uci commit lte
```

On this firmware, `ipv6='2'` makes the LTE init script start:

```text
quectel-CM -6 -s jionet
```

Apply the profile once:

```sh
/etc/init.d/lte restart
sleep 30
```

Messages such as `ifconfig: usb0 ... Device not found` can appear while the script selects QMI mode. Judge the result from the final process, link, address and route checks below.

## 3. Keep SA/NSA capability enabled

The tested router worked with automatic RAT selection and 5G enabled:

```sh
ubus -t 8 call lteat send '{"cmd":"AT+QNWPREFCFG=\"mode_pref\",AUTO"}'
ubus -t 8 call lteat send '{"cmd":"AT+QNWPREFCFG=\"nr5g_disable_mode\",0"}'
```

Do not force a cell lock unless you have recorded a known-good cell and know how to remove the lock. Network availability, tower policy and SIM provisioning still control whether SA attaches.

Optional n78-only band preference, if deliberately required:

```sh
ubus -t 8 call lteat send '{"cmd":"AT+QNWPREFCFG=\"nr5g_band\",78"}'
```

To inspect rather than change the current values:

```sh
ubus -t 8 call lteat send '{"cmd":"AT+QNWPREFCFG=\"mode_pref\""}'
ubus -t 8 call lteat send '{"cmd":"AT+QNWPREFCFG=\"nr5g_disable_mode\""}'
ubus -t 8 call lteat send '{"cmd":"AT+QNWPREFCFG=\"nr5g_band\""}'
```

## 4. Verify the modem session

```sh
ps w | grep -E '[l]teat|[q]uectel-CM'
ubus -t 10 call lteat send '{"cmd":"AT+QENG=\"servingcell\""}'
ubus -t 8 call lteat send '{"cmd":"AT+CGACT?"}'
ubus -t 8 call lteat send '{"cmd":"AT+CGPADDR=1"}'
ubus -t 8 call lteat send '{"cmd":"AT+CGCONTRDP=1"}'
```

A successful SA attachment reports `NR5G-SA`, `TDD` and band `78`. `CGACT: 1,1` means CID 1 is active, and `CGPADDR=1` should return a `2409:` IPv6 address.

`NOCONN` in `QENG` is not automatically a failure; it commonly describes the radio activity state. Check the PDP address and real connectivity as well.

## 5. Verify the OpenWrt interface and IPv6 route

```sh
ip link show dev wwan0_1
ip -6 addr show dev wwan0_1 scope global
ip -6 route show default
ping6 -c 4 2405:200:800::11
```

Expected conditions:

- `wwan0_1` is `UP,LOWER_UP`.
- At least one global `2409:` address is present.
- An IPv6 default route exists on `wwan0_1`.
- IPv6 ping succeeds.

## 6. Verify automatic 464XLAT

Jio's tested PDP is IPv6-only. OpenWrt creates a NAT46/464XLAT interface so IPv4-only clients and destinations continue to work.

```sh
uci -q get network.lte06.iface_464xlat
ps w | grep '[4]64xlat'
ip link show | grep '464-'
ping -c 4 1.1.1.1
```

The tested system showed:

```text
network.lte06.iface_464xlat='1'
464xlatcfg 464-lte06_4 wwan0_1 192.0.0.1
```

If native IPv6 works but IPv4 does not, diagnose the `lte06_4`/464XLAT interface rather than changing Jio to an IPv4 APN.

## 7. Avoid the observed reconnect conflict

The stock `wtnetcheck` service repeatedly dropped and recreated the cellular link during testing. If it causes link flaps, disable it before diagnosing the modem:

```sh
uci set wtnetcheck.main.switch='0'
uci set wtnetcheck.main.enable='0'
uci commit wtnetcheck
/etc/init.d/wtnetcheck stop
/etc/init.d/wtnetcheck disable
```

Confirm that it is stopped:

```sh
ps w | grep '[w]tnetcheck'
uci show wtnetcheck
```

## 8. Advanced modem repair used during Jio recovery

Use this section only when the normal `jionet`/IPv6 profile is saved but the modem still does not register or create CID 1. These commands change modem-persistent state and can temporarily disconnect the network.

### Inspect the selected MBN profile first

```sh
ubus -t 10 call lteat send '{"cmd":"AT+QMBNCFG=\"List\""}'
```

The recovered router showed `ROW_Commercial` as the selected/active profile. If it is already selected, do not select it again.

### Disable automatic MBN selection and select ROW

Only when the list shows the wrong carrier profile:

```sh
ubus -t 10 call lteat send '{"cmd":"AT+QMBNCFG=\"AutoSel\",0"}'
ubus -t 10 call lteat send '{"cmd":"AT+QMBNCFG=\"Select\",\"ROW_Commercial\""}'
```

Recheck the list after the modem/router restart required by your module firmware:

```sh
ubus -t 10 call lteat send '{"cmd":"AT+QMBNCFG=\"List\""}'
```

> [!CAUTION]
> MBN names and availability vary by modem firmware. Never select a profile that does not appear in `QMBNCFG="List"`. A wrong carrier profile can break registration, IMS or data service.

### Recreate Jio CID 1 as IPv6

Inspect the existing PDP contexts before changing them:

```sh
ubus -t 8 call lteat send '{"cmd":"AT+CGDCONT?"}'
```

For Jio SA, CID 1 was repaired with:

```sh
ubus -t 8 call lteat send '{"cmd":"AT+CGDCONT=1,\"IPV6\",\"jionet\""}'
```

### Test NR-only registration

For a temporary Jio SA diagnostic test:

```sh
ubus -t 8 call lteat send '{"cmd":"AT+QNWPREFCFG=\"mode_pref\",NR5G"}'
ubus -t 8 call lteat send '{"cmd":"AT+COPS=0"}'
sleep 20
ubus -t 8 call lteat send '{"cmd":"AT+C5GREG?"}'
ubus -t 10 call lteat send '{"cmd":"AT+QENG=\"servingcell\""}'
```

`C5GREG` registration state `1` (home) or `5` (roaming) indicates registered service. Restore the normal multi-operator setting after the test:

```sh
ubus -t 8 call lteat send '{"cmd":"AT+QNWPREFCFG=\"mode_pref\",AUTO"}'
```

Do not leave `mode_pref=NR5G` when Airtel NSA may be used: NSA requires an LTE anchor. The stable Jio profile documented above also uses `AUTO`; NR5G-only is a diagnostic step, not the default.

## Quick status bundle

```sh
uci -q get lte.main.apn
uci -q get lte.main.ipv6
ps w | grep -E '[l]teat|[q]uectel-CM|[4]64xlat'
ubus -t 10 call lteat send '{"cmd":"AT+QENG=\"servingcell\""}'
ubus -t 8 call lteat send '{"cmd":"AT+CGPADDR=1"}'
ip -6 addr show dev wwan0_1 scope global
ip -6 route show default
ping6 -c 4 2405:200:800::11
ping -c 4 1.1.1.1
```

Signal and speed vary with congestion, antenna placement, tower configuration and time of day. A speed test alone does not prove whether SA, routing or 464XLAT is configured correctly.
