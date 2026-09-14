# ZX7981PG 5G CPE Documentation

Community documentation for the **ZX7981PG** 5G CPE based on the MediaTek MT7981 platform and a Quectel cellular modem.

This repository brings together the tested UART/TFTP firmware recovery procedure, Jio 5G SA configuration, Airtel 5G NSA checks and automatic APN/PDP profile switching for SIM1/SIM2.

> [!CAUTION]
> This is an independent community project, not official vendor or OpenWrt support. Hardware revisions and modem firmware can differ. Back up the router configuration and factory/calibration data before making changes.

## Tested hardware and modem identification

These are the values returned by the actual recovered/tested unit, not values inferred from the product name:

| Item | Device-reported value | AT command |
|---|---|---|
| CPE product | `ZX7981PG` | Router product/UI identification |
| Modem manufacturer | `Quectel` | `AT+CGMI` |
| Modem model | `RG502Q-EU` | `ATI` / `AT+CGMM` |
| Firmware revision | `RG501QEUAAR12A01M4G_OCPU_ZM` | `AT+CGMR` |
| Full firmware build | `RG501QEUAAR12A01M4G_OCPU_ZM_04.001.04.001` | `AT+QGMR` |
| Modem hardware revision | Not reported (`ERROR`) | `AT+QHWVER` is unsupported on the tested firmware |

The `RG502Q-EU` model string and `RG501Q...` firmware identifier are both reported by the same tested modem. Do not rename or “correct” either value in documentation. `AT+QHWVER` returning `ERROR` only means that command is unsupported; it does not indicate faulty hardware. Other ZX7981PG production batches may contain a different module or firmware; verify every unit with AT commands.

```sh
ubus -t 10 call lteat send '{"cmd":"ATI"}'
ubus -t 10 call lteat send '{"cmd":"AT+CGMM"}'
ubus -t 10 call lteat send '{"cmd":"AT+CGMR"}'
ubus -t 10 call lteat send '{"cmd":"AT+QGMR"}'
ubus -t 10 call lteat send '{"cmd":"AT+QHWVER"}'
```

## Guides

| Guide | Purpose |
|---|---|
| [UART/TFTP recovery](docs/UART-TFTP-Recovery.md) | Recover the router through MediaTek U-Boot using a verified firmware image |
| [Jio 5G SA setup](docs/Jio-5G-SA-Setup.md) | Apply the tested `ROW_Commercial` MBN prerequisite, then configure `jionet`, IPv6-only PDP, n78 and 464XLAT |
| [Airtel 5G NSA setup](docs/Airtel-5G-NSA-Setup.md) | Configure dual stack and distinguish LTE carrier aggregation from active NSA |
| [SIM1/SIM2 automatic profiles](docs/SIM1-SIM2-Auto-Switch.md) | Automatically apply Jio, Airtel or Vi APN/PDP settings when the active SIM changes |

## Tested operator profiles

| Operator | APN | `lte.main.ipv6` | Expected dial command |
|---|---|---:|---|
| Jio | `jionet` | `2` | `quectel-CM -6 -s jionet` |
| Airtel | `airtelgprs.com` | `1` | `quectel-CM -4 -6 -s airtelgprs.com` |
| Vi | `www` | `1` | `quectel-CM -4 -6 -s www` |

Jio was tested on **NR5G-SA TDD n78** with a native IPv6 session and OpenWrt-created 464XLAT interface for IPv4 compatibility. On the tested new/recovered CPE, Jio registration also required MBN `AutoSel=0` with `ROW_Commercial` selected; follow the Jio guide before applying APN settings. Airtel NSA requires an LTE anchor and an attached n78 secondary carrier.

## Scripts

- [`sim-profile-onboot`](scripts/sim-profile-onboot) — selects the profile for the active SIM after boot.
- [`sim-profile-live`](scripts/sim-profile-live) — detects an active SIM identity change and restarts LTE once after applying the matching profile.

The scripts change only:

- `lte.main.apnselect`
- `lte.main.apn`
- `lte.main.ipv6`

They do **not** automatically change RAT preference, MBN profile, LTE/NR bands or cell locks.

> [!IMPORTANT]
> Read the [installation and recovery notes](docs/SIM1-SIM2-Auto-Switch.md) before copying scripts to a router. Validate them on your exact firmware and keep SSH/UART recovery access available.

The repository scripts are documented reference versions and have passed shell syntax checks. Back up and compare any known-good installed version before replacing it; retest SIM1/SIM2 and every operator profile on the target router.

## Quick Jio status check

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

## Recovery firmware

The successful UART/TFTP recovery used:

```text
ZX7981PG-9.6.1-260730-143345-sysupgrade.bin
Size: 19,857,408 bytes (0x12f0000)
SHA-256: f81ffcf83610b20e733f186d7d3971f82bed8e3ed13a35d916d16b1f132c4eed
```

See the [GitHub release](https://github.com/akswap/ZX7981PG/releases/tag/ZX7981PG) and verify the checksum before flashing. Use only `mtkupgrade fw` for the documented recovery path; do not flash BL2/FIP or run raw erase commands.

## UART reference

![ZX7981PG UART connection points](images/5fcb0645-1030-45e0-8796-3f92b7337452.png)

Only connect router `TX -> adapter RX`, router `RX -> adapter TX` and `GND -> GND`. Do not connect `3V3` or `VCC`.

## Privacy and contributions

Before posting logs or screenshots, remove IMEI, IMSI, ICCID, SIM phone number, MAC addresses and public IP addresses.

When reporting a result, include the router firmware version, modem model/firmware, operator, RAT, band, relevant AT output and whether the test was performed while data traffic was active.
