# Network Audio Streaming Setup (Debian 12 Receiver & Fedora Client)

A quick guide for setting up zero-configuration network audio streaming between a **Debian 12 (Bookworm)** server acting as the audio receiver (running PipeWire with PulseAudio emulation) and a **Fedora** client using GUI tools.

---

## 🛠️ Part 1: Debian 12 Server (Receiver Setup via SSH)

Configure PipeWire on your Debian 12 server to publish sound cards over mDNS/Zeroconf and accept incoming network audio connections over TCP.

### 1. Install Required Packages
Log in via SSH and install the PipeWire audio stack, PulseAudio emulation tools, and the Avahi mDNS service:

```bash
sudo apt update && sudo apt install -y pipewire-audio pipewire-pulse wireplumber avahi-daemon pulseaudio-utils
sudo systemctl enable --now avahi-daemon
```

### 2. Configure Persistent Network Modules
Create a PipeWire-Pulse drop-in configuration to automatically load the native TCP streaming protocol and Zeroconf broadcasting on startup:

```bash
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d/

cat <<'EOF' > ~/.config/pipewire/pipewire-pulse.conf.d/zeroconf.conf
context.exec = [
    { path = "pactl" args = "load-module module-native-protocol-tcp port=4713 listen=0.0.0.0 auth-anonymous=1" }
    { path = "pactl" args = "load-module module-zeroconf-publish" }
]
EOF
```

### 3. Restart Services & Enable Linger
Restart PipeWire services to apply changes and enable user linger so audio services run headlessly without an active SSH session:

```bash
systemctl --user restart pipewire pipewire-pulse wireplumber
sudo loginctl enable-linger $USER
```

### 4. Verification

Verify that both network modules are actively loaded in PipeWire:
```bash
pactl list modules short | grep -E 'tcp|zeroconf'
```
*(You should see `module-native-protocol-tcp` and `module-zeroconf-publish` listed).*

Verify that PipeWire is actively listening on TCP port **4713**:
```bash
ss -tulpn | grep 4713
```
*(You should see `pipewire-pulse` bound to `0.0.0.0:4713`).*

---

## 💻 Part 2: Fedora Client (GUI Setup via `paprefs`)

Configure your Fedora client desktop to discover network audio devices on the LAN using `paprefs`.

### 1. Install Discovery Backend Packages
On Fedora, `paprefs` requires the zeroconf backend module to function properly:

```bash
sudo dnf install paprefs pulseaudio-module-zeroconf
```

### 2. Restart Client Audio Engine
Restart PipeWire services on Fedora so the new module is picked up:

```bash
systemctl --user restart pipewire wireplumber
```

### 3. Enable Network Sound in GUI

1. Open **PulseAudio Preferences** (`paprefs`) from your app menu or terminal:
   ```bash
   paprefs
   ```
2. Navigate to the **Network Access** tab.
3. Check the box: **"Make discoverable PulseAudio network sound devices available locally"**.
4. Close the window.

---

## 🎧 Usage

1. Open **GNOME Settings -> Sound** (or launch `pavucontrol`) on Fedora.
2. Select your Debian 12 server's sound card from the **Output Device** menu.
3. Play any sound on Fedora—it will stream across your network and output through the Debian server's physical sound card.
