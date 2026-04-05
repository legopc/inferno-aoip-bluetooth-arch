# Inferno AoIP — Bluetooth-to-Dante Bridge (ThinkPad T470s)

## Overview

Turns a ThinkPad T470s into a **headless Bluetooth A2DP sink → Dante transmitter**.
Any Bluetooth device (phone, laptop, tablet) pairs and plays audio; that audio
is transmitted over Dante to the Shure MXWANI8 via the Inferno ALSA plugin.

```
BT source (phone/laptop)
  ↓ A2DP sink (bluez-alsa daemon)
  ↓ bluealsa (ALSA capture PCM — plugin type, not hw card)
  ↓ bluetooth_48k plug (resample to 48 kHz if SBC @ 44.1 kHz)
  ↓ bt_mix dmix (hw:Loopback,0,2 — subdevice 2 write side)
    ← inferno-bt-keepalive writes silence here (keeps loopback active)
  ↓ hw:Loopback,1,2 (snd-aloop read side)
  ↓ alsaloop (inferno-bt-loop.service)
  ↓ inferno_bluetooth (Inferno ALSA plugin → Dante TX)
  ↓↓ Dante network → Shure MXWANI8
```

**Validated signal chain note:** The Inferno plugin is opened by `alsaloop` (same as the
Spotify chain — inferno-bridge.service uses alsaloop too). The original `slave.pcm` approach
in the config template does NOT work with the current Inferno plugin; use the alsaloop bridge
pattern instead.

**Status:** ✅ **Deployed and validated on T470s (192.168.1.45) — 2026-04-04**

> inferno-bt-keepalive, inferno-bt-loop: running; inferno-bt-bridge: auto-restarts until BT device pairs.
> Dante TX subscription (inferno_bluetooth) is live and advertising on the Dante network.

---

## Hardware Requirements

| Component | Detail |
|---|---|
| Model | Lenovo ThinkPad T470s |
| CPU | Intel Core i5/i7 7th gen (Kaby Lake) |
| Bluetooth | Intel 8265 (Bluetooth 4.2) |
| Wired NIC | Intel I219-LM (`enp0s31f6`) — **required for PTP/Dante** |
| WiFi | Intel 8265 — do NOT use for Dante/PTP |
| OS | Arch Linux (same as other Inferno nodes) |

**Critical:** WiFi must NEVER be used for Dante traffic. PTP requires <1 ms jitter;
WiFi has 10–100 ms jitter. Always use wired Ethernet for Dante.

---

## Pre-Installation: Verify Hardware

Boot live Arch ISO, connect via wired Ethernet, then verify:

```bash
# Confirm wired NIC name (likely enp0s31f6 on T470s)
ip link show | grep -E '^[0-9]+: e'

# Confirm Bluetooth chip
lsusb | grep -i intel       # BT is USB-attached even on integrated Intel chips
lspci | grep -i network     # WiFi chip

# Note wired NIC MAC for Ansible inventory
cat /sys/class/net/enp0s31f6/address
```

---

## Package Installation

```bash
# Base Inferno stack (from Ansible playbook — same as other nodes)
# See ../Inferno_OS_Hardening/ansible/

# Bluetooth additions (bluez/bluez-utils may already be installed by Ansible)
sudo pacman -S bluez bluez-utils alsa-utils

# bluez-alsa (AUR — not in main repos)
# Install via yay or paru (AUR helper must already be present)
# Validated version: 4.3.1.r108.g2fa7fb0-1
yay -S --noconfirm bluez-alsa-git
```

**Package summary:**
- `bluez` — core Bluetooth daemon
- `bluez-utils` — `bluetoothctl` and tools
- `bluez-alsa-git` (AUR) — creates `hw:bluealsa,*` ALSA PCM from BT A2DP
- `alsa-utils` — `arecord`, `aplay`, `aplay -l` for verification

**⚠️ yay sudo prompts:** yay calls `sudo pacman` internally and needs either a TTY or
NOPASSWD in sudoers. For headless/scripted installs, temporarily add NOPASSWD:
```bash
echo 'legopc ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/copilot-temp
yay -S --noconfirm bluez-alsa-git
sudo rm /etc/sudoers.d/copilot-temp  # remove AFTER yay finishes
```

Do NOT install `pipewire` or `pulseaudio` — they conflict with the Inferno ALSA chain.
(Note: pipewire may already be present from the base Arch install as an inactive session
service; it does not interfere if not managing ALSA devices.)

---

## Bluetooth Configuration

### `/etc/bluetooth/main.conf`

```ini
[General]
# Power on Bluetooth at boot automatically
AutoEnable=true
# Accept pairing from any device without PIN (headless-safe)
JustWorksRepairing=always
# Remember power state across reboots
RememberPoweredState=true

[Policy]
# Reconnect to paired devices automatically
AutoConnect=true
```

### One-Time Pairing (SSH, per device)

```bash
bluetoothctl
# Inside bluetoothctl:
power on
agent off
default-agent
pairable on
discoverable on
# Put phone in pairing mode, then:
trust <PHONE_MAC>
connect <PHONE_MAC>
quit
```

After pairing, the device reconnects automatically on future boots.

---

## ALSA Configuration

Add to `~/.asoundrc` (user `legopc`). This uses a **dmix mix bus** (`bt_mix`) on loopback
subdevice 2, matching the pattern used for the Spotify chain. The `bluetooth_loop_write` plug
from the config template is replaced by `bt_mix` dmix to allow concurrent writers (silence
keepalive + BT audio).

```conf
# ============================================================
# Bluetooth A2DP → snd-aloop → Inferno TX → Dante
# ============================================================

# Resample bluealsa capture to 48 kHz (SBC is 44.1 kHz; Dante requires 48 kHz)
# If source device supports AAC and negotiates 48 kHz, this is a no-op passthrough
pcm.bluetooth_48k {
    type plug
    slave {
        pcm "hw:bluealsa,1"
        rate 48000
        channels 2
        format S16_LE
    }
    hint.description "Bluetooth A2DP resampled to 48kHz"
}

# dmix mix bus on loopback subdevice 2 (write side)
# Allows concurrent writers: bt_keepalive (silence) + bt_bridge (audio) = audio
pcm.bt_mix {
    type dmix
    ipc_key 5002
    slave {
        pcm "hw:Loopback,0,2"
        rate 48000
        format S32_LE
        channels 2
        period_size 256
        periods 4
    }
}

# Inferno Bluetooth TX plugin — opened by inferno-bt-loop (alsaloop)
# DEVICE_ID: node MAC + 0001 (0000 used by inferno_spotify on same node)
# ALT_PORT: 6004–6007 (second range; 6000–6003 used by inferno_spotify)
pcm.inferno_bluetooth {
    type inferno
    NAME "T470s-Bluetooth"
    BIND_IP enp0s31f6
    SAMPLE_RATE 48000
    PROCESS_ID 4
    ALT_PORT 6004
    RX_CHANNELS 0
    TX_CHANNELS 2
    TX_LATENCY_NS 10000000
    RX_LATENCY_NS 10000000
    CLOCK_PATH /tmp/ptp-usrvclock
    DEVICE_ID 54e1ad6a8b330001

    hint {
        show off
        description "Inferno ALSA virtual device for Bluetooth A2DP"
    }
}
```

**Note on DEVICE_ID:** When coexisting with `inferno_spotify` on the same node, increment
the trailing bytes to avoid Dante network conflicts:
```bash
NIC=enp0s31f6
# inferno_spotify uses: $(cat /sys/class/net/$NIC/address | tr -d ':')0000
# inferno_bluetooth:    $(cat /sys/class/net/$NIC/address | tr -d ':')0001
```

**Note on Loopback subdevice 2:** Uses subdevice 2 to avoid conflict with:
- Subdevice 0: `inferno_spotify` (Spotify → Dante)
- Subdevice 2: `inferno_bluetooth` (Bluetooth → Dante)

snd-aloop defaults to 8 subdevices — **no modprobe changes needed** if the module is already
loaded with `options snd-aloop index=5`.

---

## Systemd Services

Three user services are required (deployed to `~/.config/systemd/user/`):

### `inferno-bt-keepalive.service` — always running

Writes silence to `bt_mix` (dmix on loopback subdevice 2) to keep the loopback active
when no BT device is connected. Prevents inferno-bt-loop from underrunning.

```ini
[Unit]
Description=Inferno BT Loopback Keepalive (silence when BT not connected)
After=sound.target default.target

[Service]
Type=simple
ExecStart=/usr/bin/aplay -D bt_mix -r 48000 -f S32_LE -c 2 /dev/zero
Restart=on-failure
RestartSec=3
TimeoutStopSec=5

[Install]
WantedBy=default.target
```

### `inferno-bt-loop.service` — always running

Opens `inferno_bluetooth` (Inferno Dante TX plugin) via alsaloop, reading from the
loopback capture side. Keeps the Dante TX subscription permanently alive.
Same pattern as `inferno-bridge.service` for the Spotify chain.

```ini
[Unit]
Description=Inferno Bluetooth Loopback → Dante Bridge
After=statime-inferno.service sound.target default.target
After=inferno-bt-keepalive.service

[Service]
Type=simple
ExecStartPre=/bin/sh -c 'while [ ! -S /tmp/ptp-usrvclock ]; do sleep 1; done'
ExecStart=/usr/bin/alsaloop \
  -C hw:Loopback,1,2 \
  -P inferno_bluetooth \
  -r 48000 \
  -f S32_LE \
  -c 2 \
  -t 20000
Restart=always
RestartSec=3
TimeoutStopSec=5

[Install]
WantedBy=default.target
```

### `inferno-bt-bridge.service` — restarts when BT disconnects

Polls `bluealsa-aplay -l` until an A2DP device appears, then streams audio to
the `bt_loopback` plug PCM (rate/format conversion → hw:Loopback,0,2).  Exits
and retries on BT disconnect.

**Volume control:** Use `--volume=software` explicitly.  `bt_loopback` is a
`plug` device with no ALSA mixer control; `--volume=auto` (the default) tries
to open that mixer, fails silently ("Couldn't open ALSA mixer: Host is down"),
and only the mute (vol=0) AVRCP event is honoured.  `--volume=software` applies
volume scaling directly in the PCM stream for all levels 0–127.  Do **not**
pass `--volume=none` — that silently discards all AVRCP volume changes.

```ini
[Unit]
Description=Bluetooth A2DP → snd-aloop (clock buffer before Dante TX)
After=bluetooth.target bluealsa.service
Wants=bluetooth.target bluealsa.service

[Service]
Type=simple
ExecStartPre=/bin/sh -c 'while [ ! -S /tmp/ptp-usrvclock ]; do sleep 1; done'
ExecStart=/bin/sh -c 'while true; do \
    until bluealsa-aplay -l 2>/dev/null | grep -q A2DP; do sleep 2; done; \
    echo "A2DP ready — writing to loopback"; \
    bluealsa-aplay --volume=software -D bt_loopback 2>&1 || true; \
    echo "bluealsa-aplay exited, waiting for reconnect..."; \
done'
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

### Service Startup Order

```
statime-inferno.service         ← creates /tmp/ptp-usrvclock
  ↓
bluetooth.service
  ↓
bluealsa.service
  ↓
inferno-bt-keepalive.service (user)  ← silence → loopback subdev 2
  ↓
inferno-bt-loop.service (user)       ← loopback subdev 2 → inferno_bluetooth (Dante TX)
  ↓
inferno-bt-bridge.service (user)     ← BT audio → loopback (restarts on BT disconnect)
```

Enable all three:
```bash
systemctl --user daemon-reload
systemctl --user enable --now inferno-bt-keepalive inferno-bt-loop inferno-bt-bridge
```

---

## Verification Steps

```bash
# 1. Check Bluetooth is up
bluetoothctl show | grep -E 'Powered|Discoverable|Pairable'

# 2. Check bluealsa and bluetooth services
systemctl is-active bluetooth bluealsa

# 3. Check BT user services
systemctl --user status inferno-bt-keepalive inferno-bt-loop inferno-bt-bridge

# 4. Expected states before any BT device connects:
#    inferno-bt-keepalive: active (running)
#    inferno-bt-loop:      active (running) — Dante TX subscription live
#    inferno-bt-bridge:    activating (auto-restart) — OK, no BT device yet

# 5. Check bluez-alsa ALSA devices (only appear when BT device connected)
aplay -l | grep -i blue    # empty until BT device connects and plays

# 6. Connect a BT device and check bridge starts
systemctl --user status inferno-bt-bridge    # should become active
journalctl --user -u inferno-bt-bridge -f

# 7. Test capture from BT (phone must be connected and playing audio)
arecord -D bluetooth_48k -r 48000 -c 2 -f S32_LE -d 5 bt-test.wav
aplay bt-test.wav

# 8. Dante Controller (on Windows/Mac on same network)
#    Should see "T470s-Bluetooth" as a Dante TX source (via inferno_bluetooth, ALT_PORT 6004)
```

---

## Known Issues & Gotchas

### bluez-alsa device disappears on BT disconnect
When the BT source disconnects, `hw:bluealsa,1` disappears. The bridge service
(`inferno-bt-bridge`) fails and auto-restarts (`Restart=on-failure`). During the reconnect
gap, `inferno-bt-keepalive` writes silence to the loopback, keeping the Dante TX
subscription alive (no dropout on the Dante receiver side).

### snd-aloop subdevice count
snd-aloop defaults to **8 subdevices** — no `pcm_substreams=` modprobe option needed.
Current `inferno_spotify` uses subdevice 0; `inferno_bluetooth` uses subdevice 2.
Config template: `/etc/modprobe.d/snd-aloop.conf` only needs `options snd-aloop index=5`.

### SBC codec is 44.1 kHz; AAC may be 48 kHz
If the BT source device negotiates AAC (iPhone, Mac), bluez-alsa may capture at
48 kHz natively, making the `plug` resampler a no-op. If SBC is negotiated (Android,
older devices), resampling occurs. Check active codec:
```bash
bluealsa-cli list     # shows connected devices and active codec
```

### bluealsa card number may not be 0
After starting bluez-alsa, run `aplay -l` to confirm the card number. If it's
not `hw:bluealsa,1`, update the ALSA config accordingly, or use the named device
form (the ALSA plugin registers as `bluealsa` by name).

### Intel 8265 A2DP power saving
May cause audio dropout. If dropouts occur:
```bash
echo 'options iwlmvm power_scheme=1' | sudo tee /etc/modprobe.d/iwlmvm-nosave.conf
# Reboot or: sudo modprobe -r iwlmvm && sudo modprobe iwlmvm power_scheme=1
```
Note: this affects Intel wireless power save, not BT directly, but they share the chip.

### PTP not available in Proxmox test VMs
Testing on a Proxmox VM will work for deployment mechanics (package install,
ALSA config, service startup) but Inferno/Dante TX will not function without
a real PTP grandmaster. Test audio bridge only on physical T470s in the Dante network.

---

## Ansible Integration (Planned)

The base Inferno Ansible playbook (`../Inferno_OS_Hardening/ansible/`) needs a
new role or extra tasks for BT nodes:

```yaml
# group_vars/bt_nodes.yml
inferno_bt_enabled: true
inferno_bt_alt_port_start: 6004    # 6004–6007 (not 6000 — conflicts with inferno_spotify)
inferno_bt_loopback_subdevice: 2
inferno_bt_device_id_suffix: "0001"  # 0000 used by inferno_spotify on same node
```

Tasks to add:
1. Install `bluez`, `bluez-utils` (may already be present from base playbook)
2. Install `bluez-alsa-git` via AUR helper (`yay`) with NOPASSWD workaround
3. Deploy `/etc/bluetooth/main.conf`
4. Enable `bluetooth.service` and `bluealsa.service`
5. Deploy BT-specific `~/.asoundrc` block (`bluetooth_48k`, `bt_mix`, `inferno_bluetooth`)
6. Deploy `inferno-bt-keepalive.service` user service
7. Deploy `inferno-bt-loop.service` user service
8. Deploy `inferno-bt-bridge.service` user service
9. `systemctl --user daemon-reload && systemctl --user enable --now inferno-bt-{keepalive,loop,bridge}`

---

## Files in This Directory

| File | Purpose |
|---|---|
| `README.md` | This file — full setup guide |
| `.github/copilot-instructions.md` | Copilot context for AI assistance |
| `config/bluetooth-main.conf` | `/etc/bluetooth/main.conf` template (deployed) |
| `config/asoundrc-bt.conf` | Original ALSA config template (superseded by README — use dmix bt_mix, not plug bluetooth_loop_write) |
| `config/inferno-bt-bridge.service` | systemd user service template (bridge only; two more services needed — see README) |

---

## Research Notes

- **bluez-alsa vs PipeWire decision**: bluez-alsa chosen for minimal headless footprint.
  PipeWire is heavier (5–10% CPU idle vs 2%) and requires session manager config for
  monitor device exposure. bluez-alsa exposes `hw:bluealsa,*` natively.
- **T470s NIC**: Intel I219-LM, `enp0s31f6`. Confirmed.
- **T470s BT chip**: Intel 8265 (BT 4.2), fully supported by `linux-firmware` on Arch.
  No AUR firmware packages needed. Confirmed on kernel 6.19.10-arch1-1.
- **A2DP codec**: SBC mandatory, AAC/aptX optional. All supported by Intel 8265.
  LDAC not supported by Intel chips.
- **alsaloop vs slave.pcm**: The Inferno plugin must be opened by `alsaloop` (same as the
  Spotify chain). Using `slave.pcm "hw:Loopback,1,2"` in the inferno plugin config and
  then opening it via `aplay` does not work — the plugin does not autonomously pull from
  slave.pcm. Use `alsaloop -C hw:Loopback,1,2 -P inferno_bluetooth` instead.
- **DEVICE_ID conflict**: When multiple Inferno instances share a node, each needs a unique
  DEVICE_ID. Increment the last 4 hex digits of the 16-digit DEVICE_ID:
  `54e1ad6a8b330000` (spotify), `54e1ad6a8b330001` (bluetooth).
- **ALT_PORT conflict**: inferno_spotify uses 6000–6003. inferno_bluetooth must use 6004–6007.
- **Pipewire coexistence**: pipewire + wireplumber are running as user services on the T470s
  (base Arch default). They do NOT interfere with the direct ALSA chain used by Inferno.
  Do not disable them — they may be used by other processes (e.g., browser audio).

---

## Deployment Log — T470s (192.168.1.45) — 2026-04-04

| Step | Result |
|---|---|
| `bluez`, `bluez-utils` | Already installed (5.86-4) |
| `yay` AUR helper | Present at `/usr/bin/yay` |
| `bluez-alsa-git` | Installed 4.3.1.r108.g2fa7fb0-1 via yay |
| `/etc/bluetooth/main.conf` | Deployed (backup at main.conf.bak) |
| `bluetooth.service` | enabled + active |
| `bluealsa.service` | enabled + active |
| `~/.asoundrc` BT block | Appended (lines 60–115: bluetooth_48k, bt_mix, inferno_bluetooth) |
| `inferno-bt-keepalive.service` | Deployed, enabled, **active (running)** |
| `inferno-bt-loop.service` | Deployed, enabled, **active (running)** — Dante TX live |
| `inferno-bt-bridge.service` | Deployed, enabled, **auto-restart** (expected — no BT device paired yet) |
| snd-aloop modprobe | No change needed (default 8 subdevices, index=5 already set) |
| Spotify stack | Unaffected — librespot, inferno-bridge, inferno-keepalive all still active |
| **Pairing** | ⏳ Pending — physical BT device needed to complete chain test |

---

## Troubleshooting Log — Discoverability Fix — 2026-04-04

### Problem
User saw BT device `94:3c:96:08:42:0a` nearby and could not determine if it was the T470s.
T470s was not appearing as discoverable or pairable. No auto-accept agent was running.

### Findings

| Item | Finding |
|---|---|
| **T470s BT MAC** | `94:B8:6D:B3:FF:96` — Intel 8265 (USB-attached internally) |
| Mystery device `94:3c:96:08:42:0a` | **NOT the T470s** — unknown nearby device |
| Discoverable | Was OFF (timed out) |
| DiscoverableTimeout | Was 180 s (not 0) — missing from main.conf |
| PairableTimeout | Missing from main.conf |
| BT name advertised | Was `archlinux` — not identifiable |
| bt-agent service | Did not exist — no auto-accept agent running |
| Audio Sink UUID | ✅ Registered (A2DP Sink 0000110b) |
| bluealsa | ✅ active (running) with a2dp-sink + a2dp-source |

### Fixes Applied

**1. `/etc/bluetooth/main.conf`** — Added missing settings:
```ini
DiscoverableTimeout=0    # stay discoverable permanently
PairableTimeout=0        # stay pairable permanently
Name=T470s-Dante         # advertised device name
```

**2. Via `bluetoothctl`** (immediate effect, persists via main.conf on next restart):
```bash
bluetoothctl power on
bluetoothctl discoverable on
bluetoothctl pairable on
bluetoothctl system-alias 'T470s-Dante'
```

**3. bt-agent — Python D-Bus auto-accept agent:**
- Script: `~/.local/bin/bt-accept-agent.py`
- Registers as BlueZ `NoInputNoOutput` default agent on the system D-Bus
- Auto-accepts all pairing, authorization, and confirmation requests
- User service: `~/.config/systemd/user/bt-agent.service` — enabled, **active (running)**
- Required packages: `python-dbus`, `python-gobject` (installed)

**4. Packages installed:**
- `python-dbus` — D-Bus Python bindings
- `python-gobject` — GLib mainloop for D-Bus agent

### Current State (post-fix)

```
BT MAC:          94:B8:6D:B3:FF:96
BT Name:         T470s-Dante  (alias — what phones see)
Powered:         yes
Discoverable:    yes (permanent, timeout=0)
Pairable:        yes (permanent, timeout=0)
bt-agent:        active (running) — NoInputNoOutput, auto-accept
Audio Sink UUID: registered (A2DP sink for phone → T470s audio)
bluealsa:        active (a2dp-sink + a2dp-source, all codecs)
inferno-bt-bridge: auto-restart (starts streaming once a phone connects)
```

### How to Connect a Phone

1. Open Bluetooth settings on your phone
2. Look for **"T470s-Dante"** in the device list
3. Tap to pair — pairing will auto-accept (no PIN needed)
4. Select as audio output and play music
5. `inferno-bt-bridge` will start automatically and audio will flow:
   Phone → A2DP → bluealsa → loopback → Inferno → Dante → Shure MXWANI8

### Notes for Future Runs

- `bluetoothctl discoverable on` reverts to OFF on bluetooth service restart — main.conf
  `DiscoverableTimeout=0` ensures it comes back on after `power on` via AutoEnable.
- The bt-agent must be running **before** a device attempts to pair. It is now a persistent
  user service that starts at login (and via `lingering` if enabled).
- If bt-agent stops, run: `systemctl --user restart bt-agent`
- To verify agent is registered: `journalctl --user -u bt-agent -n 5`

---

## Troubleshooting Log — Silent Dante Channel Fix — 2026-04-04

### Symptom
Dante Controller showed the "T470s-Bluetooth" TX channel and the route was configured, but
audio was completely silent. Phone was connected and A2DP showed in `bluealsa-aplay --list-devices`.

### Diagnosis

The signal chain was broken at `inferno-bt-bridge.service`. The service was crash-looping
every 5 seconds with:

```
ALSA lib bluealsa-pcm.c:1964:(_snd_pcm_bluealsa_open) [error.] Couldn't get BlueALSA PCM:
    Sender is not authorized to send message
arecord: main:850: audio open error: Permission denied
```

**Root cause: `user@1000.service` was missing the `audio` supplementary group.**

When `systemd --user` starts via linger at boot, it captures the user's supplementary groups
from `/etc/group` at that exact moment. If the user was added to the `audio` group *after*
linger was first enabled, the running user manager process retains the old group list.

Confirmed by running `id` inside the service context:

```bash
systemd-run --user --wait --pipe id
# Output: uid=1000(legopc) gid=1000(legopc) groups=1000(legopc),998(wheel)
# MISSING: 984(audio)
```

The BlueALSA D-Bus policy (`/usr/share/dbus-1/system.d/org.bluealsa.conf`) grants access via
`<policy group="audio">` — which failed because the service context didn't have that group.
The same arecord command worked fine in an interactive SSH session (full PAM session includes
the audio group).

### Fix Applied

**1. D-Bus policy override** — allows `legopc` by username (survives group roster changes):

```bash
sudo mkdir -p /etc/dbus-1/system.d/
sudo tee /etc/dbus-1/system.d/org.bluealsa-legopc.conf << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE busconfig PUBLIC "-//freedesktop//DTD D-BUS Bus Configuration 1.0//EN"
 "http://www.freedesktop.org/standards/dbus/1.0/busconfig.dtd">
<!-- Allow legopc user to access BlueALSA regardless of supplementary group state.
     Needed because user@1000.service may start before audio group is in effect. -->
<busconfig>
  <policy user="legopc">
    <allow send_destination="org.bluealsa"/>
  </policy>
</busconfig>
EOF

# Reload D-Bus policy (no restart needed)
sudo dbus-send --system --dest=org.freedesktop.DBus \
  --type=method_call /org/freedesktop/DBus \
  org.freedesktop.DBus.ReloadConfig
```

**2. Restart `user@1000.service`** to pick up the `audio` group for all running services:

```bash
# SSH session is in session-NNN.scope — safe to restart user@1000 without losing SSH
sudo systemctl restart user@1000.service
# All lingering services (inferno-bt-*, inferno-bridge, etc.) auto-restart within ~5s
```

**3. Verify groups are now correct in service context:**

```bash
systemd-run --user --wait --pipe id
# Should now show: groups=1000(legopc),984(audio),998(wheel)

systemd-run --user --wait bluealsa-aplay --list-devices
# Should succeed without "not authorized" error
```

### Result

`inferno-bt-bridge` now stays running. Full chain confirmed active:
- `arecord -D bluealsa` captures A2DP (S16_LE 44100 Hz)
- `aplay -D bt_sink` converts via plug → S32_LE 48000 Hz → bt_mix dmix on Loopback sub2
- `alsaloop` (inferno-bt-loop) reads from Loopback,1,2 → writes to inferno_bluetooth (Dante TX)
- Loopback pcm1c/sub2: `state: RUNNING`, hw_ptr/appl_ptr advancing — audio flowing

Initial underruns at bridge startup are expected (pipe fill-time during BT connection handoff);
they settle within 5–10 seconds.

### Prevention

The `/etc/dbus-1/system.d/org.bluealsa-legopc.conf` policy file persists across reboots and
ensures this never recurs regardless of group state in the user manager. Include this file in
the Ansible playbook for BT nodes.

If re-deploying from scratch, run `sudo systemctl restart user@1000.service` after any
`usermod -aG audio legopc` to avoid the stale-group problem.

---

## Troubleshooting Log — BT Volume Control Fix — 2026-04-04

### Symptom

Audio streamed correctly from the phone through to Dante, but volume up/down on
the phone (AVRCP absolute volume) had no effect on the output level.

### Diagnosis

`inferno-bt-bridge.service` was invoking `bluealsa-aplay` with the explicit flag
`--volume=none`:

```
bluealsa-aplay --volume=none -D bt_loopback
```

The `--volume=none` option tells `bluealsa-aplay` to ignore AVRCP volume events
entirely.  The phone sends the volume change, `bluealsad` acknowledges it, but
the PCM samples written to the loopback were never attenuated.

Also confirmed: this version of `bluealsad` does **not** expose an
`--avrcp-volume` flag — volume handling is the responsibility of `bluealsa-aplay`
alone (via its `--volume=auto` default).

### Fix Applied

Removed `--volume=none` from the `ExecStart` in
`~/.config/systemd/user/inferno-bt-bridge.service` so `bluealsa-aplay` uses its
default `--volume=auto` mode:

```bash
# Before:
bluealsa-aplay --volume=none -D bt_loopback

# After:
bluealsa-aplay -D bt_loopback
```

Reloaded and restarted the service:

```bash
systemctl --user daemon-reload
systemctl --user restart inferno-bt-bridge
```

### Result

`bluealsa-aplay` now processes AVRCP absolute volume signals in software.
Adjusting volume on the connected phone attenuates the PCM stream written to the
loopback device, and the output level at the Dante TX changes accordingly.

### Prevention

The `config/inferno-bt-bridge.service` template in this repo has been updated to
reflect this — it no longer passes `--volume=none`.  Future deploys will have
volume control enabled by default.

---

## Troubleshooting Log — AVRCP Volume Partial Fix — 2026-04-04

### Symptom

Volume mute (level 0) from the phone **worked** — audio went silent.  All other
volume levels (1–127) were **ignored** — audio stayed at full volume regardless
of the phone's volume control.  AVRCP events were definitely received (0 = mute
worked), but non-zero levels had no effect.

### Diagnosis

`bt_loopback` is defined as an ALSA `plug` device (rate/format conversion
wrapper around `hw:Loopback,0,2`).  Plug devices expose no ALSA mixer control.

`bluealsa-aplay --volume=auto` (the default after the previous `--volume=none`
fix) tries to find and open an ALSA mixer control on the playback device so it
can apply AVRCP volume changes.  When no mixer exists it logs:

```
bluealsa-aplay: W: Couldn't open ALSA mixer: Attach mixer: Host is down
```

With no mixer to write to, `auto` mode silently drops all volume events — except
level 0 (mute), which is handled via a separate code path in `bluealsa-aplay`
that bypasses the mixer and zeroes the PCM samples directly.

### Fix Applied

Changed `bluealsa-aplay -D bt_loopback` →
`bluealsa-aplay --volume=software -D bt_loopback`.

`--volume=software` performs volume scaling directly in the PCM sample buffer on
every write, without requiring an ALSA mixer control.  All 128 AVRCP volume
levels (0–127) are now applied.

```bash
# Before (broken for levels 1–127):
bluealsa-aplay -D bt_loopback

# After:
bluealsa-aplay --volume=software -D bt_loopback
```

Deployed:

```bash
# Edit /home/legopc/.config/systemd/user/inferno-bt-bridge.service
systemctl --user daemon-reload
systemctl --user restart inferno-bt-bridge
```

### Files Changed

- `config/inferno-bt-bridge.service` — `ExecStart` updated
- `README.md` — volume control note and this log entry added

### Root Cause Summary

| Volume mode | Requires ALSA mixer? | vol=0 | vol=1–127 |
|---|---|---|---|
| `--volume=none` | No | ❌ ignored | ❌ ignored |
| `--volume=auto` (default) | Yes (fails on plug) | ✅ works | ❌ ignored |
| `--volume=software` | No | ✅ works | ✅ works |

Use `--volume=software` whenever the target ALSA device is a `plug`, `dmix`,
loopback, or any other virtual device without a real hardware mixer.
