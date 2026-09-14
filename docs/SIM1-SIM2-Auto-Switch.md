# SIM1/SIM2 Automatic Operator Profile Switching

The ZX7981PG can use different operators in SIM1 and SIM2. Changing the active slot in the web UI changes the modem's SIM, but the saved APN/PDP profile may still belong to the previous operator. The scripts in [`../scripts`](../scripts) address this by detecting the active SIM and applying the matching persistent profile.

## Profile map

| Detected operator | APN | `lte.main.ipv6` | Dial mode |
|---|---|---:|---|
| Jio | `jionet` | `2` | IPv6 only (`-6`) |
| Airtel | `airtelgprs.com` | `1` | IPv4 + IPv6 (`-4 -6`) |
| Vi | `www` | `1` | IPv4 + IPv6 (`-4 -6`) |

The scripts leave `mode_pref`, LTE/NR bands, cell locks and `nr5g_disable_mode` untouched.

## Script roles

- `sim-profile-onboot`: checks the active SIM after boot, saves the matching UCI profile and reboots once only when the persistent profile changed.
- `sim-profile-live`: watches for an active SIM identity change, including a change between two SIMs on the same operator, then applies the matching profile and restarts LTE once.

The live watcher does not implement a continuous PDP reconnect loop. It does not repeatedly restart a broken network connection.

## Install

Copy both scripts to the router:

```powershell
scp -O .\scripts\sim-profile-onboot root@192.168.88.1:/tmp/
scp -O .\scripts\sim-profile-live root@192.168.88.1:/tmp/
```

Then, over SSH:

```sh
cp /tmp/sim-profile-onboot /usr/bin/sim-profile-onboot
cp /tmp/sim-profile-live /usr/bin/sim-profile-live
chmod 755 /usr/bin/sim-profile-onboot /usr/bin/sim-profile-live
sh -n /usr/bin/sim-profile-onboot
sh -n /usr/bin/sim-profile-live
```

Create `/etc/init.d/sim-profile-onboot`:

```sh
#!/bin/sh /etc/rc.common
START=98
STOP=10

start() {
    /usr/bin/sim-profile-onboot >/dev/null 2>&1 &
}

stop() {
    rm -f /tmp/sim-profile-onboot.lock
}
```

Create `/etc/init.d/sim-profile-live`:

```sh
#!/bin/sh /etc/rc.common
START=99
STOP=9
USE_PROCD=1

start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/sim-profile-live
    procd_set_param respawn 3600 5 5
    procd_close_instance
}
```

Enable both services:

```sh
chmod 755 /etc/init.d/sim-profile-onboot /etc/init.d/sim-profile-live
/etc/init.d/sim-profile-onboot enable
/etc/init.d/sim-profile-live enable
/etc/init.d/sim-profile-live start
```

## Verify

```sh
/usr/bin/sim-profile-onboot --status
logread -e sim-profile-onboot
logread -e sim-profile-live
ps w | grep -E '[s]im-profile-(onboot|live)'
uci -q get lte.main.apn
uci -q get lte.main.ipv6
ps w | grep '[q]uectel-CM'
```

After a slot change, allow time for SIM registration, operator detection, one LTE restart and PDP setup. Jio should finally show `quectel-CM -6 -s jionet`; Airtel/Vi should show `-4 -6` with their respective APN.

## Recovery from an AT-command collision

Never start multiple manual copies of the selector. If several copies or blocked `ubus call lteat` processes exist:

```sh
/etc/init.d/sim-profile-live stop
/etc/init.d/sim-profile-onboot stop 2>/dev/null
for p in $(ps w | awk '$0 ~ /\/usr\/bin\/sim-profile-(onboot|live)/ && $0 !~ /awk/ {print $1}'); do kill -9 "$p"; done
for p in $(ps w | awk '$0 ~ /ubus .*call lteat/ && $0 !~ /awk/ {print $1}'); do kill -9 "$p"; done
rm -f /tmp/sim-profile-onboot.lock /tmp/sim-profile-live.lock
/etc/init.d/lte restart
```

Wait about 30 seconds before testing `ubus -t 5 call lteat send '{"cmd":"AT"}'`.

