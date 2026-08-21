---
tags:

- troubleshooting
- networking
- linux
- wifi

---

# WiFi Card Drops Connection Under Load Due to PCIe Power Management (mt7925e Forced Reassociation)

Even after fixing WiFi bufferbloat (see
[Related Articles](#related-articles)), a follow-up session under sustained load still
hit a real outage — not jitter, an actual dropped connection — caused by the WiFi card's
PCIe power management confusing its own firmware into forcing a reassociation.

!!! note
    This fix is not yet considered fully proven — it's held up over the sessions tested
    so far, but the underlying driver/firmware behavior hasn't been root-caused down to
    the silicon level. This article will be updated as more is learned.

## Environment

- OS: Ubuntu (kernel `6.17.0-1032-oem`)
- WiFi chipset: MediaTek MT7925E (`mt7925e` driver, PCIe, device `0000:04:00.0`)
- Network manager: NetworkManager
- Found during the same investigation as
  [WiFi bufferbloat](wifi-bufferbloat-lan-streaming-drops.md), with `fq_codel` already
  applied and confirmed active

## Problem

With `fq_codel` fixed and verified active, a follow-up Expo session still hit a real
outage — not jitter this time, actual `Destination Host Unreachable` for ~13 seconds,
followed by a kernel-logged forced reassociation:

```text
wlp4s0: deauthenticating from 2e:70:4f:18:e8:d8 by local choice (Reason: 3=DEAUTH_LEAVING)
wlp4s0: authenticate with 2e:70:4f:18:e8:d8 (try 1/3)
wlp4s0: associated
```

The radio recovered for about 17 seconds (clean 1-4ms pings), then degraded back into
sustained heavy jitter (100ms-1500ms) that didn't fully resolve for the rest of the
session — worse than plain bufferbloat, and `tc qdisc show` confirmed `fq_codel` was
still active throughout, so queue management wasn't the issue this time.

## Root Cause

`by local choice` means the *driver* initiated the disconnect, not the router. This is a
known class of instability on the MediaTek `mt7925e` chipset (and several other WiFi
chipsets): under sustained load, the driver/firmware can get confused by the PCIe link
transitioning through power-saving states (ASPM L0s/L1) or by PCIe runtime autosuspend
kicking in, and forces a reassociation to recover — sometimes coming back in a degraded
state rather than cleanly.

```bash
$ cat /sys/module/pcie_aspm/parameters/policy
default [performance] powersave powersupersave    # was previously "default" (active)

$ cat /sys/bus/pci/devices/0000:04:00.0/power/control
on                                                 # was "auto" (runtime autosuspend allowed)
```

Checked first whether a newer firmware fixed it — it didn't apply here: `linux-firmware`
was already at Ubuntu's latest packaged version, and `fwupdmgr` doesn't manage this
chip's firmware (it ships as part of the kernel firmware package, not as a
`fwupd`-updatable device). So the fix targets the power-management trigger directly
instead of waiting on an upstream firmware update.

## Fix

**Important — scope matters here.** `/sys/module/pcie_aspm/parameters/policy` is a
**global** kernel setting: it affects every PCIe device (NVMe SSD, GPU, everything), not
just WiFi. Setting it to `performance` system-wide measurably increases idle power draw
and heat (worse battery life, more fan noise), especially with a discrete GPU in the
picture. Prefer the WiFi-scoped version below unless the machine is always plugged in.

**Recommended — scoped to the WiFi card only:**

```bash
# Disable PCIe runtime autosuspend for just the WiFi device
sudo sh -c 'echo on > /sys/bus/pci/devices/0000:04:00.0/power/control'

# Disable ASPM (L0s/L1 low-power link states) on just the WiFi device's PCIe link,
# by clearing bits 0-1 of its Link Control register (CAP_EXP+0x10) — leaves every
# other PCIe device's power management untouched
sudo setpci -s 04:00.0 CAP_EXP+10.w=0000:0003
```

Verify:

```bash
$ cat /sys/bus/pci/devices/0000:04:00.0/power/control
on

$ sudo setpci -s 04:00.0 CAP_EXP+10.w
0040   # low nibble even (0, 4, 8, or c) = ASPM bits clear = disabled
```

**To reverse (restore default power-saving behavior on the WiFi card):**

```bash
sudo sh -c 'echo auto > /sys/bus/pci/devices/0000:04:00.0/power/control'
sudo setpci -s 04:00.0 CAP_EXP+10.w=0003:0003
```

**If a global test is wanted instead** (simpler, but see the battery/heat caveat above):

```bash
# Apply
sudo sh -c 'echo performance > /sys/module/pcie_aspm/parameters/policy'

# Reverse
sudo sh -c 'echo default > /sys/module/pcie_aspm/parameters/policy'
```

Swap `0000:04:00.0` / `04:00.0` for your own WiFi device's PCI address
(`lspci | grep -i network`) if different.

Both the scoped commands above are **runtime-only** — they reset on reboot. Persisting
them (e.g. via a `systemd-tmpfiles` rule or a small `systemd` oneshot service run at
boot, since this isn't tied to a NetworkManager interface-up event the way `fq_codel`
is) is a reasonable next step once confirmed stable over real use, but hold off making
it permanent until it's proven itself across more than one session.

## Prevention

- **Recognize the pattern**: this is a *different* symptom from bufferbloat — an actual
  dropped connection (`Destination Host Unreachable`) with a kernel-logged
  `deauthenticating ... by local choice` and reassociation, not just latency/jitter. It
  can also show up even when `fq_codel` is already confirmed active, which rules out
  queueing as the cause.
- **Quick diagnostic**: `journalctl -k -f` during the outage — `by local choice` means
  the driver initiated the disconnect itself, pointing at the driver/firmware/power
  stack rather than the router or ISP.
- **Check the power-management state**: `cat /sys/module/pcie_aspm/parameters/policy`
  and `cat /sys/bus/pci/devices/<wifi-pci-addr>/power/control` — if ASPM is active
  (`default`) or runtime autosuspend is allowed (`auto`), that's the first thing to rule
  out on this class of instability.
- **This is not proven fully solved** — treat it as a working mitigation to keep
  watching, not a closed case.

## Related Articles

- [WiFi Bufferbloat Collapses the Connection When Streaming to a LAN Device](wifi-bufferbloat-lan-streaming-drops.md) —
  the other bug found on the same machine during this investigation; fixing it does not
  rule out this one.
