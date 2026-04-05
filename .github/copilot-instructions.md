# Copilot Instructions — Inferno AoIP Bluetooth-to-Dante

## What This Project Is

A sub-project that turns a **ThinkPad T470s into a headless Bluetooth-to-Dante bridge**.
Any Bluetooth A2DP source (phone, laptop, tablet) pairs with the T470s and plays audio.
That audio is captured and transmitted to the Dante network via the Inferno AoIP ALSA
plugin → Shure MXWANI8.

**Hardware target:** Lenovo ThinkPad T470s
- CPU: Intel Core i5/i7 (7th gen, Kaby Lake)
- BT/WiFi: Intel 8265 (Bluetooth 4.2, Wi-Fi 5)
- NIC: Intel I219-LM (wired, used for PTP/Dante)
- Audio: Realtek ALC298 (onboard, not used in this project)
- OS: Arch Linux (same deployment as other Inferno nodes)

## Relationship to Parent Projects

Read in order:
1. `../Inferno_AoIP/.github/copilot-instructions.md` — base setup (Arch, ALSA, PTP, Inferno)
2. `../Inferno_AoIP_AUX/.github/copilot-instructions.md` — AUX TX mode (this project is similar but BT source)

The Inferno ALSA plugin, Statime PTP daemon, and systemd service patterns are all
inherited from the parent project. The only new components are:
- `bluez-alsa` — creates ALSA PCM device from Bluetooth A2DP sink
- `inferno-bt-bridge.service` — pipes bluealsa capture → snd-aloop → Inferno ALSA plugin

## Signal Chain

```
Bluetooth device (phone/laptop)
  ↓ A2DP sink (bluez / bluez-alsa daemon)
  ↓ hw:bluealsa,1 (bluez-alsa ALSA capture PCM)
  ↓ plug / speexrate (resample to 48 kHz if codec rate ≠ 48000)
  ↓ hw:Loopback,0,2 (snd-aloop write subdevice — device 2 for BT)
  ↓ hw:Loopback,1,2 (snd-aloop read subdevice)
  ↓ inferno_bluetooth (Inferno ALSA plugin, TX-only to Dante)
  ↓↓ Dante network → Shure MXWANI8
```

## DEVICE_ID for This Node

Derive from wired NIC MAC (NOT WiFi/BT adapter MACs):
```bash
NIC=enp0s31f6  # T470s typical wired NIC — verify with: ip link show | grep -E '^[0-9]+: e'
DEVICE_ID=$(cat /sys/class/net/$NIC/address | tr -d ':')0000
```

For BT-TX instance specifically, use trailing byte 02 if other instances exist on same node:
`<mac_without_colons>0002`

## ALT_PORT Range for BT Instance

If this node runs ONLY the BT Inferno instance: use `6000–6003`.
If coexisting with Spotify or AUX on the same node: use `6012–6015`.

## Key Design Decisions

### bluez-alsa vs PipeWire
**Use bluez-alsa.** Reasons:
- Exposes native `hw:bluealsa,1` ALSA capture device — no extra routing
- Minimal daemon with no desktop/display requirements
- Proven compatible with snd-aloop + Inferno chain
- Low CPU overhead (~2% idle vs 5–10% for PipeWire)
- No PipeWire graph complexity needed for headless node

### Auto-Accept Pairing (Headless)
Configure `/etc/bluetooth/main.conf`:
```ini
[General]
AutoEnable=true
JustWorksRepairing=always
[Policy]
AutoConnect=true
```
Initial pairing via `bluetoothctl` (one-time, SSH):
```bash
bluetoothctl -- trust <PHONE_MAC>
```

### Resampling
Bluetooth A2DP SBC codec → 44.1 kHz. Inferno/Dante requires 48 kHz.
Use ALSA `plug` type with explicit `rate 48000` slave, or `speexrate` plugin.
Insert plug between `hw:bluealsa,1` and the loopback write side.

## Critical Unknowns (resolve before finalising)

1. **bluez-alsa AUR package name**: `bluez-alsa-git` (AUR) vs `bluez-alsa` — confirm available
2. **T470s wired NIC name**: typically `enp0s31f6` but verify with `ip link show`
3. **bluealsa card number**: after daemon starts, run `aplay -l` and note card/device numbers
4. **A2DP codec negotiation**: SBC (44.1 kHz) vs AAC/aptX (48 kHz native) — if source device
   supports AAC, bluez-alsa can negotiate 48 kHz and skip resampling
5. **Does bluez-alsa PCM stay open when BT device disconnects?** Need keepalive or reconnect logic

## Keepalive Strategy

When the BT device disconnects, `hw:bluealsa,1` disappears. The bridge service must
handle this gracefully:
- Option A: Bridge service restarts on failure (`Restart=on-failure`) — simple
- Option B: Dedicated watchdog script that holds loopback open with silence and
  restarts bridge on reconnect — avoids Dante TX dropout on reconnect

For initial implementation, use Option A.

## Ansible Integration

The Ansible playbook at `../Inferno_OS_Hardening/ansible/` deploys the base Inferno
stack. BT support should be added as an optional role or group_vars flag:

```yaml
# group_vars/bluetooth_nodes.yml
inferno_bt_enabled: true
inferno_bt_alt_port: 6000
inferno_bt_device_id_suffix: "0000"
```

New tasks needed:
- Install `bluez-alsa-git` (AUR) via AUR helper
- Enable `bluetooth.service` + `bluealsa.service`
- Write `/etc/bluetooth/main.conf`
- Add bt-bridge systemd user service
- Add BT-specific ALSA config block to `~/.asoundrc`
- Add `inferno_bluetooth` Inferno plugin instance

## Status

**Phase**: Research complete, implementation not yet started.

See `README.md` for full setup instructions once implementation is validated.
