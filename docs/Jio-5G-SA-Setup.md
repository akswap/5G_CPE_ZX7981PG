# Jio 5G SA Setup for ZX7981PG

This guide records the configuration successfully used with a Jio SIM on the ZX7981PG. The observed network was **NR5G-SA**, band **n78**, with Jio's IPv6-only PDP service and automatic 464XLAT for IPv4 destinations.

> [!IMPORTANT]
> Back up `/etc/config/lte` and `/etc/config/network` before changing anything. Commands and interface names in this guide are specific to the tested firmware. Do not publish screenshots containing IMEI, IMSI, ICCID, phone number, MAC address or public IP.

> [!WARNING]
> **New or recovered ZX7981PG units may not register on Jio until the modem MBN is corrected.** On the tested firmware, Jio registration started only after disabling MBN automatic selection and selecting `ROW_Commercial`. Complete section 3 before changing the APN/PDP settings. MBN inventories vary by modem firmware, so first confirm that `ROW_Commercial` exists in your own list.

## Working profile

| Setting | Tested value |
|---|---|
| Operator | Jio (`405/871`) |
| MBN automatic selection | `0` (disabled) |
| Selected MBN | `ROW_Commercial` |
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

## 2. Open SSH and send AT commands

Connect a PC to a router LAN port, open **Windows PowerShell** and run:

```powershell
ssh root@192.168.88.1
```

Accept the host-key prompt only when you are sure this is your directly connected router, then enter the router's root password. A successful login shows a prompt similar to:

```text
root@OpenWrt:~#
```

After a firmware recovery, Windows may still have the old SSH key. Confirm the router address and then remove only that saved key:

```powershell
ssh-keygen -R 192.168.88.1
ssh root@192.168.88.1
```

On this firmware, send modem AT commands through the running `lteat` service. First confirm that it is ready:

```sh
ps w | grep '[l]teat'
ubus -t 5 call lteat send '{"cmd":"AT"}'
```

The second command should return `OK`. The reusable format is:

```sh
ubus -t 10 call lteat send '{"cmd":"AT_COMMAND"}'
```

For example, the raw modem command `AT+QMBNCFG="List"` becomes:

```sh
ubus -t 10 call lteat send '{"cmd":"AT+QMBNCFG=\"List\""}'
```

Quotes inside the AT command must be escaped as `\"`. Run one AT request at a time and wait for its result. Do not write directly to `/dev/ttyUSB2` while `lteat` is running; two processes competing for the modem port can cause timeouts and hide the IMEI/SIM details in the web UI. If `AT` times out immediately after boot, wait 20–30 seconds and try once more.

## 3. Set the required MBN profile first

This is the key one-time registration fix on the tested new/recovered CPE. An APN change alone will not help if the modem has selected an incompatible carrier MBN.

### 3.1 Inspect the available and selected MBN

```sh
ubus -t 10 call lteat send '{"cmd":"AT+QMBNCFG=\"List\""}'
ubus -t 10 call lteat send '{"cmd":"AT+QMBNCFG=\"AutoSel\""}'
```

Confirm that `ROW_Commercial` appears in the list. In the `List` response, the selected profile is the entry whose selected flag is `1`. If `ROW_Commercial` is already selected and `AutoSel` is already `0`, do not write the settings again.

### 3.2 Disable AutoSel and select ROW_Commercial

Run these commands only if the checks above show a different state:

```sh
ubus -t 10 call lteat send '{"cmd":"AT+QMBNCFG=\"AutoSel\",0"}'
ubus -t 10 call lteat send '{"cmd":"AT+QMBNCFG=\"Select\",\"ROW_Commercial\""}'
```

Restart the router once so the modem reloads the selected MBN:

```sh
reboot
```

After reconnecting by SSH, verify the selection again:

```sh
ubus -t 10 call lteat send '{"cmd":"AT+QMBNCFG=\"List\""}'
ubus -t 10 call lteat send '{"cmd":"AT+QMBNCFG=\"AutoSel\""}'
```

> [!CAUTION]
> Never select an MBN name that is absent from your modem's `QMBNCFG="List"` output. A wrong MBN can break network registration, IMS or data service. The `ROW_Commercial` result documented here applies to the tested ZX7981PG modem firmware.

Raw AT commands used in this step, for users working in another proper modem AT console:

| Purpose | Raw AT command |
|---|---|
| List available/selected MBN profiles | `AT+QMBNCFG="List"` |
| Read the AutoSel state | `AT+QMBNCFG="AutoSel"` |
| Disable automatic MBN selection | `AT+QMBNCFG="AutoSel",0` |
| Select the tested profile | `AT+QMBNCFG="Select","ROW_Commercial"` |

## 4. Save the Jio APN and IPv6-only mode

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

## 5. Keep SA/NSA capability enabled

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

## 6. Verify registration and the modem session

First request automatic network selection and check 5G registration:

```sh
ubus -t 8 call lteat send '{"cmd":"AT+COPS=0"}'
sleep 20
ubus -t 8 call lteat send '{"cmd":"AT+C5GREG?"}'
```

`C5GREG` state `1` (home) or `5` (roaming) indicates registered service.

```sh
ps w | grep -E '[l]teat|[q]uectel-CM'
ubus -t 10 call lteat send '{"cmd":"AT+QENG=\"servingcell\""}'
ubus -t 8 call lteat send '{"cmd":"AT+CGACT?"}'
ubus -t 8 call lteat send '{"cmd":"AT+CGPADDR=1"}'
ubus -t 8 call lteat send '{"cmd":"AT+CGCONTRDP=1"}'
```

A successful SA attachment reports `NR5G-SA`, `TDD` and band `78`. `CGACT: 1,1` means CID 1 is active, and `CGPADDR=1` should return a `2409:` IPv6 address.

`NOCONN` in `QENG` is not automatically a failure; it commonly describes the radio activity state. Check the PDP address and real connectivity as well.

## 7. Verify the OpenWrt interface and IPv6 route

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

## 8. Verify automatic 464XLAT

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

## 9. Avoid the observed reconnect conflict

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

## 10. Registration/PDP recovery checks

Use this section only when the required MBN is selected and the normal `jionet`/IPv6 profile is saved, but the modem still does not register or create CID 1. These commands change modem-persistent state and can temporarily disconnect the network.

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
