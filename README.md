# adsb-config

Configuration and scripts for a FlightAware ADS-B receiver running on a Raspberry Pi with an RTL-SDR dongle (RTL2832U chipset).

## The problem

The RTL2832U chip hangs after extended operation — typically many hours — with USB errors `-71` or `-110`. When this happens, `dump1090-fa` enters a crash loop emitting `no supported devices found`, the dongle stops receiving aircraft, and piaware stops feeding FlightAware. The only fix is to reset or re-enumerate the USB device.

## rtlsdr-watchdog

`usr/local/bin/rtlsdr-watchdog` runs every 5 minutes via a systemd timer and detects the hang by counting `no supported devices found` errors in the `dump1090-fa` journal over the past 5 minutes. If it sees 3 or more, it attempts recovery in order:

1. **uhubctl power cycle** — cuts power to the USB hub port the dongle is on, waits, restores power, and checks whether the device re-enumerates. This is a full electrical reset and the most reliable fix.
2. **sysfs authorized toggle** — if uhubctl fails or isn't available, toggles `/sys/bus/usb/devices/1-2/authorized` to force a soft USB reset without cutting power.
3. **Full reboot** — if neither software method recovers the device, reboots the host as a last resort. The journal is flushed (`journalctl --sync`) before rebooting so the failure is recorded.

A lock file (`/var/lock/rtlsdr-watchdog.lock`) prevents concurrent instances from racing when the timer fires while a previous recovery attempt is still in progress.

## Files

| File | Installed path |
|------|---------------|
| `usr/local/bin/rtlsdr-watchdog` | `/usr/local/bin/rtlsdr-watchdog` |
| `etc/systemd/system/rtlsdr-watchdog.service` | `/etc/systemd/system/rtlsdr-watchdog.service` |
| `etc/systemd/system/rtlsdr-watchdog.timer` | `/etc/systemd/system/rtlsdr-watchdog.timer` |

## Installation

```bash
sudo cp usr/local/bin/rtlsdr-watchdog /usr/local/bin/rtlsdr-watchdog
sudo chmod +x /usr/local/bin/rtlsdr-watchdog
sudo cp etc/systemd/system/rtlsdr-watchdog.* /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now rtlsdr-watchdog.timer
```

Requires `uhubctl` for the power-cycle path (`sudo apt install uhubctl`). The sysfs fallback works without it.

## Usage

Check whether the timer is active and when it last/next ran:

```bash
systemctl status rtlsdr-watchdog.timer
```

View the watchdog's full log history:

```bash
journalctl -t rtlsdr-watchdog
```

View only recovery attempts and reboots:

```bash
journalctl -t rtlsdr-watchdog | grep -v "exit 0"
```

Run the watchdog manually (useful for testing):

```bash
sudo /usr/local/bin/rtlsdr-watchdog
```
