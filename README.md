# Inferno AoIP — Bluetooth A2DP → Dante Bridge (Arch Linux)

Turns a headless Arch Linux machine with Bluetooth into a **Bluetooth A2DP sink → Dante transmitter**. Any phone, laptop, or tablet pairs and plays audio; that audio is transmitted over Dante to any Dante receiver on the network.

```
BT source (phone / laptop / tablet)
  ↓ A2DP sink (bluez-alsa daemon)
  ↓ bluealsa ALSA capture PCM
  ↓ bt_sink plug (resample + format convert: S16_LE 44.1 kHz → S32_LE 48 kHz)
  ↓ bt_mix dmix (snd-aloop write side, loopback subdevice 2)
    ← inferno-bt-keepalive writes silence here (holds loopback open)
  ↓ hw:Loopback,1,2 (snd-aloop read side)
  ↓ inferno-bt-loop (alsaloop bridge)
  ↓ inferno_bluetooth (Inferno ALSA plugin → Dante TX)
  ↓↓ Dante network → any Dante receiver
```

**Status:** ✅ Deployed and validated on an Intel I219-LM Ethernet + Intel 8265 BT machine.

> **⚠️ Key finding:** The Inferno plugin must be opened by `alsaloop`, not via `slave.pcm`. The loopback bridge pattern (`snd-aloop` → `alsaloop` → Inferno) is required — see Architecture section.

---

## Hardware Requirements

| Requirement | Notes |
|-------------|-------|
| Any x86-64 machine | Laptop, mini PC, thin client |
| Bluetooth adapter | Intel 8265 validated; most Linux-supported adapters work |
| Wired Ethernet | **Required** — never use WiFi for Dante/PTP |
| Arch Linux | Rolling release; tested on current stable |

> **⚠️ WiFi must NEVER carry Dante traffic.** PTP requires <1 ms jitter. WiFi has 10–100 ms jitter. Use wired Ethernet for the Dante NIC even on a laptop.

> **⚠️ If Bluetooth and Ethernet share the same chip** (Intel 8265 combo card): Intel wireless power saving can cause audio dropout. If this occurs, disable power saving: `echo 'options iwlmvm power_scheme=1' | sudo tee /etc/modprobe.d/iwlmvm-nosave.conf`

---

## Prerequisites — Inferno Stack

Each repo is self-contained. Set up the full stack from scratch:

### 1. System packages

```bash
sudo pacman -Syu
sudo pacman -S pipewire pipewire-alsa alsa-utils alsa-lib base-devel git speexdsp alsa-plugins \
               bluez bluez-utils
systemctl --user enable --now pipewire wireplumber
sudo systemctl enable --now bluetooth
```

### 2. Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

### 3. AUR helper + bluez-alsa

```bash
cd ~ && git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si

# bluez-alsa is not in the main repos — install from AUR
# Validated version: bluez-alsa-git 4.3.1.r108+
yay -S bluez-alsa-git
```

> **⚠️ yay sudo prompts:** For headless/scripted installs, temporarily add NOPASSWD, install, then remove:
> ```bash
> echo "$USER ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/yay-temp
> yay -S --noconfirm bluez-alsa-git
> sudo rm /etc/sudoers.d/yay-temp
> ```

> **⚠️ Do NOT install `pipewire-pulse` or `pulseaudio`** — they intercept Bluetooth audio before bluez-alsa can. The base `pipewire` + `wireplumber` (without pulse) is fine and does not interfere with the direct ALSA chain.

### 4. Disable system time sync (REQUIRED)

```bash
sudo systemctl mask systemd-timesyncd.service
```

> **⚠️ Critical:** Statime manages the system clock. Any competing NTP/PTP daemon conflicts with it. Keep time sync masked permanently.

### 5. Build Statime (PTP clock daemon with PTPv1)

```bash
cd ~
git clone --recurse-submodules -b inferno-dev https://github.com/teodly/statime
cd statime && cargo build
```

> **⚠️ Fork required:** Standard upstream Statime only supports PTPv2. Dante hardware devices use PTPv1. You **must** use the `inferno-dev` fork.

Edit `inferno-ptpv1.toml` — set `interface` to your wired NIC name:
```bash
cp config/statime/inferno-ptpv1.toml ~/statime/inferno-ptpv1.toml
# Edit: change interface = "enp1s0" to your wired NIC name (ip link show | grep '^[0-9].*e')
```

### 6. Statime system service

```bash
sudo cp config/systemd-system/statime-inferno.service /etc/systemd/system/
# Edit the service file: replace 'legopc' with your username, verify statime path
sudo systemctl enable --now systemd-networkd-wait-online.service
sudo systemctl daemon-reload
sudo systemctl enable --now statime-inferno.service
```

### 7. Build and install Inferno ALSA plugin

```bash
cd ~
git clone --recurse-submodules https://gitlab.com/lumifaza/inferno.git
cd inferno

# PipeWire syscall allowance
mkdir -p ~/.config/systemd/user/pipewire.service.d/
cp os_integration/systemd_allow_clock.conf \
   ~/.config/systemd/user/pipewire.service.d/override.conf
systemctl --user daemon-reload && systemctl --user restart pipewire

# Build
cd alsa_pcm_inferno && cargo build --release

# Install
sudo mkdir -p /usr/lib/alsa-lib
sudo cp ../target/release/libasound_module_pcm_inferno.so /usr/lib/alsa-lib/
```

### 8. Register plugin type and load snd-aloop

```bash
sudo cp config/alsa/99-inferno.conf /etc/alsa/conf.d/99-inferno.conf

echo 'options snd-aloop index=5' | sudo tee /etc/modprobe.d/snd-aloop.conf
sudo modprobe snd-aloop
echo 'snd-aloop' | sudo tee /etc/modules-load.d/snd-aloop.conf
```

### 9. Add user to audio group + enable linger

```bash
sudo usermod -aG audio $USER
sudo loginctl enable-linger $USER
```

> **⚠️ Audio group stale state:** When `loginctl enable-linger` is active, the user manager starts at boot before the `audio` group change is reflected. If services fail with BlueALSA "not authorized" errors, deploy the D-Bus policy override (`config/dbus/org.bluealsa-USERNAME.conf`) and restart `user@UID.service` — see Troubleshooting section.

---

## Compute Your DEVICE_ID

```bash
NIC=eth0   # replace with your wired NIC name
MAC=$(cat /sys/class/net/$NIC/address | tr -d ':')
echo "${MAC}0000"   # for inferno_spotify if coexisting
echo "${MAC}0001"   # for inferno_bluetooth
```

> **⚠️ DEVICE_IDs must be unique** across your entire Dante network. Derive from actual MAC.

---

## Bluetooth Configuration

### `/etc/bluetooth/main.conf`

```bash
sudo cp config/bluetooth-main.conf /etc/bluetooth/main.conf
sudo systemctl restart bluetooth
```

Key settings in the config:
- `AutoEnable=true` — powers Bluetooth on at boot
- `DiscoverableTimeout=0` — stays discoverable permanently
- `PairableTimeout=0` — stays pairable permanently
- `Name=BT-Dante` — the name phones see when scanning (change to your preference)
- `JustWorksRepairing=always` — accepts pairing without PIN (headless-safe)
- `AutoConnect=true` — reconnects paired devices automatically

### One-Time Pairing (per device, via SSH)

```bash
bluetoothctl
# Inside bluetoothctl interactive shell:
power on
discoverable on
pairable on
# Put phone in BT pairing mode, then:
pair <PHONE_MAC>    # or just wait — with the bt-agent, pairing is auto-accepted
trust <PHONE_MAC>
quit
```

After initial pairing, the device reconnects automatically on future boots via `AutoConnect=true`.

### Auto-Accept Agent (headless pairing)

A Python D-Bus agent (`bt-agent`) handles pairing requests without requiring interactive input. Install required packages and the agent service:

```bash
sudo pacman -S python-dbus python-gobject

# Create the agent script
mkdir -p ~/.local/bin
cp config/bt-agent.py ~/.local/bin/bt-accept-agent.py
chmod +x ~/.local/bin/bt-accept-agent.py

# Install and enable the user service
cp config/systemd-user/bt-agent.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now bt-agent.service
```

With the agent running, any new device can pair by simply selecting your machine from Bluetooth settings — no PIN or confirmation needed.

---

## ALSA Configuration

Edit `config/asoundrc-bt.conf` and replace placeholders:

| Placeholder | Replace with |
|-------------|-------------|
| `YOUR_NIC` | wired NIC name (`ip link show`) |
| `YOUR_BT_DEVICE_ID` | MAC + 0001 (see DEVICE_ID section) |
| `YOUR_BT_DEVICE_NAME` | name in Dante Controller (e.g. `Office-Bluetooth`) |

Then append to `~/.asoundrc`:

```bash
cat config/asoundrc-bt.conf >> ~/.asoundrc
```

> **⚠️ PROCESS_ID and ALT_PORT:** If running Spotify TX alongside Bluetooth on the same machine, ensure PROCESS_ID and ALT_PORT ranges don't conflict:
> - Spotify: PROCESS_ID 1, ALT_PORT 6000
> - Bluetooth: PROCESS_ID 4, ALT_PORT 6004 (or any other non-overlapping range)

> **⚠️ snd-aloop subdevice:** Uses subdevice 2 (`hw:Loopback,0,2`) to avoid conflict with the Spotify chain (subdevice 0). Default snd-aloop provides 8 subdevices — no extra modprobe options needed.

---

## Systemd Services

Three user services are required. Deploy them all:

```bash
mkdir -p ~/.config/systemd/user
cp config/inferno-bt-keepalive.service ~/.config/systemd/user/
cp config/inferno-bt-loop.service ~/.config/systemd/user/
cp config/inferno-bt-bridge.service ~/.config/systemd/user/
# Edit each: replace 'legopc' with your username if paths reference home dir
systemctl --user daemon-reload
systemctl --user enable --now inferno-bt-keepalive inferno-bt-loop inferno-bt-bridge
```

### Service roles

| Service | Role | Expected state (no BT device connected) |
|---------|------|-----------------------------------------|
| `inferno-bt-keepalive` | Writes silence to loopback — keeps snd-aloop and Dante TX alive | `active (running)` |
| `inferno-bt-loop` | Permanent alsaloop: loopback → Inferno → Dante TX | `active (running)` |
| `inferno-bt-bridge` | Polls until A2DP device appears, then streams audio | `activating` (restart loop) |

The `inferno-bt-bridge` restart loop is **expected** when no BT device is connected. Once a device connects and starts A2DP, it transitions to `active (running)`.

### Service startup order

```
statime-inferno.service                ← creates /tmp/ptp-usrvclock
  ↓
bluetooth.service + bluealsa.service
  ↓
inferno-bt-keepalive.service (user)    ← silence → loopback subdev 2
  ↓
inferno-bt-loop.service (user)         ← loopback subdev 2 → Dante TX (always running)
  ↓
inferno-bt-bridge.service (user)       ← BT audio → loopback (starts when phone connects)
```

---

## Verification

```bash
# 1. Check Statime is syncing
sudo systemctl status statime-inferno
journalctl -u statime-inferno | grep -i offset | tail -5

# 2. Check BT and bluealsa system services
systemctl is-active bluetooth bluealsa

# 3. Check user services
systemctl --user status inferno-bt-keepalive inferno-bt-loop inferno-bt-bridge

# 4. Check Dante TX is advertising
avahi-browse _netaudio-arc._tcp --terminate
# → Should see inferno_bluetooth device

# 5. Connect a phone and check bridge activates
systemctl --user status inferno-bt-bridge
journalctl --user -u inferno-bt-bridge -f

# 6. Test BT capture (phone must be connected and playing)
arecord -D bluealsa -r 44100 -c 2 -f S16_LE -d 5 bt-test.wav && aplay bt-test.wav
```

---

## Known Issues & Caveats

### bluez-alsa device disappears on BT disconnect
When the BT source disconnects, the bluealsa PCM disappears. `inferno-bt-bridge` exits and auto-restarts (restart loop). During the gap, `inferno-bt-keepalive` writes silence to the loopback, keeping the Dante TX subscription alive — no dropout on the receiver side.

### BlueALSA "not authorized" — audio group stale state
**Symptom:** `inferno-bt-bridge` fails with "Sender is not authorized to send message" or "Permission denied".

**Root cause:** When linger is enabled, `user@UID.service` starts at boot before supplementary group changes (like adding to `audio`) are applied. The BlueALSA D-Bus policy grants access by group membership.

**Fix:** Deploy the D-Bus policy override to grant access by username instead:
```bash
# Rename org.bluealsa-legopc.conf to use your actual username
sudo cp config/dbus/org.bluealsa-legopc.conf /etc/dbus-1/system.d/org.bluealsa-$(whoami).conf
sudo sed -i "s/legopc/$(whoami)/g" /etc/dbus-1/system.d/org.bluealsa-$(whoami).conf

# Reload D-Bus policy (no restart needed)
sudo dbus-send --system --dest=org.freedesktop.DBus \
  --type=method_call /org/freedesktop/DBus \
  org.freedesktop.DBus.ReloadConfig

# Restart user manager to pick up audio group
sudo systemctl restart user@$(id -u).service
```

### SBC vs AAC codec affects sample rate
- SBC (Android, most devices): 44.1 kHz S16_LE — resampling occurs via `bt_sink` plug
- AAC (iPhone, Mac): 48 kHz possible — resampler becomes a no-op passthrough

Check active codec:
```bash
bluealsa-cli list   # shows connected devices and negotiated codec
```

### Volume control — use `--volume=software`
`bluealsa-aplay --volume=auto` (default) tries to find an ALSA mixer control on the playback device. `bt_sink` is a `plug` device with no mixer, so auto-mode silently drops all AVRCP volume events except mute.

`--volume=software` (used in `inferno-bt-bridge.service`) applies volume scaling directly in the PCM sample buffer — all 128 AVRCP volume levels work correctly. Do not change this to `--volume=none` or `--volume=auto`.

### PTP not available without a real Dante network
Inferno/Dante TX will not function without a PTP grandmaster on the network. Test audio bridging only on machines connected to a physical Dante network.

---

## Firewall Ports (UDP, all inbound)

| Port | Purpose |
|------|---------|
| 4455 | Dante control |
| 8700 | Dante media |
| 4400 | Dante control alt |
| 8800 | Dante media alt |
| 5353 | mDNS (Dante device discovery) |
| 6004–6007 | Inferno ALT_PORT range (Bluetooth instance) |

---

## Troubleshooting

```bash
# Check all user services
systemctl --user status inferno-bt-keepalive inferno-bt-loop inferno-bt-bridge

# Check what groups the user manager sees
systemd-run --user --wait --pipe id

# Check bluealsa devices (only visible when BT device is connected)
bluealsa-aplay --list-devices 2>/dev/null || echo "No A2DP devices"

# Check Dante TX advertising
avahi-browse _netaudio-arc._tcp --terminate

# Check Bluetooth state
bluetoothctl show | grep -E 'Powered|Discoverable|Pairable|Alias'

# Tail bridge log during phone connection
journalctl --user -u inferno-bt-bridge -f

# Test bluealsa access from service context
systemd-run --user --wait --pipe bluealsa-aplay --list-devices

# Check PTP socket
ls -la /tmp/ptp-usrvclock

# Check Statime clock offset
journalctl -u statime-inferno | grep -i "offset\|sync" | tail -10
```

---

## References

- Inferno: https://gitlab.com/lumifaza/inferno (canonical), https://github.com/teodly/inferno (mirror)
- Statime fork: https://github.com/teodly/statime branch `inferno-dev`
- bluez-alsa: https://github.com/arkq/bluez-alsa
- Spotify → Dante setup: https://github.com/legopc/inferno-aoip-spotify-arch
- AUX → Dante setup: https://github.com/legopc/inferno-aoip-aux-arch
