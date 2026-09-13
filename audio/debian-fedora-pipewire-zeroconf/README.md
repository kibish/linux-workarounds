# Network Audio Streaming Setup (Debian 12 Receiver & Fedora Client)

A quick guide for setting up network audio streaming between a **Debian
12 (Bookworm)** server acting as the audio receiver and a **Fedora**
desktop acting as the client.

Both systems use **PipeWire with PulseAudio compatibility**.

This guide was tested with:

-   **Client:** Fedora Linux 43 (Workstation Edition)
-   **Receiver:** Debian GNU/Linux 12 (Bookworm)

The Debian server publishes its audio devices over mDNS/Zeroconf. Fedora
can either discover these devices automatically or connect to a specific
remote sink manually.

------------------------------------------------------------------------

## 🛠️ Part 1: Debian 12 Server (Receiver)

Configure PipeWire on the Debian server to accept PulseAudio-compatible
TCP connections and advertise its audio devices via Avahi/mDNS.

### 1. Install Required Packages

``` bash
sudo apt update
sudo apt install -y pipewire-audio pipewire-pulse wireplumber \
    avahi-daemon pulseaudio-utils

sudo systemctl enable --now avahi-daemon
```

Verify that PulseAudio compatibility is provided by PipeWire:

``` bash
pactl info | grep "Server Name"
```

Expected output should look similar to:

``` text
Server Name: PulseAudio (on PipeWire ...)
```

### 2. Configure Persistent Network Modules

Create a PipeWire-Pulse configuration drop-in:

``` bash
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d/

cat <<'EOF' > ~/.config/pipewire/pipewire-pulse.conf.d/zeroconf.conf
context.exec = [
    { path = "pactl" args = "load-module module-native-protocol-tcp port=4713 listen=0.0.0.0 auth-anonymous=1" }
    { path = "pactl" args = "load-module module-zeroconf-publish" }
]
EOF
```

> **Security note:** `auth-anonymous=1` allows clients that can reach
> TCP port 4713 to connect without PulseAudio authentication. Use this
> only on a trusted LAN and restrict access with a firewall if
> necessary.

### 3. Restart Services and Enable Linger

``` bash
systemctl --user restart pipewire pipewire-pulse wireplumber

sudo loginctl enable-linger "$USER"
```

### 4. Verification

Check that the modules are loaded:

``` bash
pactl list modules short | grep -E 'native-protocol-tcp|zeroconf'
```

You should see:

``` text
module-native-protocol-tcp
module-zeroconf-publish
```

Check that TCP port 4713 is listening:

``` bash
ss -lntp | grep 4713
```

You should see `pipewire-pulse` listening on port `4713`.

Check the advertised sinks:

``` bash
avahi-browse -rt _pulse-sink._tcp
```

For sources:

``` bash
avahi-browse -rt _pulse-source._tcp
```

------------------------------------------------------------------------

## 💻 Part 2: Fedora Client

Fedora uses PipeWire by default. Do **not** install the traditional
`pulseaudio` daemon or `pulseaudio-module-zeroconf` for this setup.

First verify that PulseAudio compatibility is provided by PipeWire:

``` bash
pactl info | grep "Server Name"
```

Expected:

``` text
Server Name: PulseAudio (on PipeWire ...)
```

If `pactl` reports plain `pulseaudio`, check your Fedora audio
configuration before continuing.

### Method A: Automatic Zeroconf Discovery

This method automatically discovers PulseAudio/PipeWire sinks and
sources advertised on the LAN.

First make sure Avahi is available:

``` bash
sudo dnf install avahi avahi-tools
sudo systemctl enable --now avahi-daemon
```

Verify that the Debian server is visible:

``` bash
avahi-browse -rt _pulse-sink._tcp
```

You should see the Debian server's advertised sinks.

Create the PipeWire Zeroconf discovery configuration:

``` bash
mkdir -p ~/.config/pipewire/pipewire.conf.d/

cat <<'EOF' > ~/.config/pipewire/pipewire.conf.d/zeroconf-discover.conf
context.modules = [
    {
        name = libpipewire-module-zeroconf-discover
        args = { }
    }
]
EOF
```

Restart the client audio stack:

``` bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

Check the discovered devices:

``` bash
wpctl status
```

The remote sinks should now appear under `Audio -> Sinks`.

You can also verify the generated tunnel nodes:

``` bash
pw-cli ls Node | grep 'tunnel\.'
```

#### ⚠️ Bluetooth / A2DP Note

On some PipeWire/WirePlumber configurations, enabling network audio
tunnels may interact badly with Bluetooth audio.

A symptom is that a Bluetooth headset which normally provides an A2DP
**High Fidelity Playback** profile suddenly exposes only HSP/HFP
(**Handsfree**) profiles.

Check available Bluetooth profiles with:

``` bash
pactl list cards
```

If A2DP disappears after enabling Zeroconf discovery, disable the
configuration:

``` bash
mv ~/.config/pipewire/pipewire.conf.d/zeroconf-discover.conf \
   ~/.config/pipewire/pipewire.conf.d/zeroconf-discover.conf.disabled

systemctl --user restart pipewire pipewire-pulse wireplumber
```

Note that a manually created Pulse tunnel may also trigger the same
problem on affected systems. Therefore Method B should not be considered
a guaranteed workaround for the Bluetooth/A2DP issue.

------------------------------------------------------------------------

### Method B: Connect to a Remote Sink Manually

If automatic discovery is unnecessary, create a tunnel directly to a
specific sink on the Debian server.

This avoids automatically importing every advertised source and sink and
is useful when you only need one specific remote output.

#### 1. Find the Remote Sink Name

On the Debian server:

``` bash
pactl list sinks short
```

Example:

``` text
alsa_output.usb-Creative_Technology_SB_X-Fi_Surround_5.1__blank_-00.stereo-fallback
```

#### 2. Test the Tunnel

On Fedora:

``` bash
pactl load-module module-tunnel-sink \
    server=tcp:192.168.178.70:4713 \
    sink=alsa_output.usb-Creative_Technology_SB_X-Fi_Surround_5.1__blank_-00.stereo-fallback
```

Replace:

-   `192.168.178.70` with the Debian server's address.
-   `sink=` with the sink name returned by `pactl list sinks short`.

Check:

``` bash
wpctl status
```

The remote sink should now appear as an output device.

To unload a tunnel created for testing, note the numeric module ID
returned by `pactl load-module` and run:

``` bash
pactl unload-module <MODULE_ID>
```

#### 3. Make the Manual Tunnel Persistent

Create a PipeWire-Pulse configuration:

``` bash
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d/

cat <<'EOF' > ~/.config/pipewire/pipewire-pulse.conf.d/remote-sink.conf
pulse.cmd = [
    {
        cmd = "load-module"
        args = "module-tunnel-sink server=tcp:192.168.178.70:4713 sink=alsa_output.usb-Creative_Technology_SB_X-Fi_Surround_5.1__blank_-00.stereo-fallback"
    }
]
EOF
```

Then restart:

``` bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

------------------------------------------------------------------------

## 🎧 Usage

Open:

**GNOME Settings → Sound**

or:

``` bash
pavucontrol
```

The Debian server's audio device should appear in the output device
list.

Select it and play audio normally. Applications on Fedora will send
their audio through PipeWire over the network to the Debian server.

------------------------------------------------------------------------

## 🔧 Troubleshooting

### Remote devices are not discovered

Check mDNS discovery on Fedora:

``` bash
avahi-browse -rt _pulse-sink._tcp
```

If the server appears here but not in `wpctl status`, the problem is on
the PipeWire discovery/tunnel side rather than Avahi.

### Check the network connection directly

``` bash
nc -vz 192.168.178.70 4713
```

Replace the address with the Debian server's IP address.

### Check remote tunnel nodes

``` bash
pw-cli ls Node | grep tunnel
```

### Check PipeWire logs

``` bash
journalctl --user -u pipewire -b | \
    grep -iE 'zeroconf|tunnel|error|fail'
```

### Bluetooth headset stuck in Handsfree mode

Inspect the Bluetooth card:

``` bash
pactl list cards
```

A working stereo Bluetooth headset should normally expose one or more
A2DP profiles such as:

``` text
a2dp-sink
a2dp-sink-sbc
a2dp-sink-sbc_xq
```

If only profiles such as these remain:

``` text
headset-head-unit
headset-head-unit-cvsd
```

disable the network-audio configuration, restart PipeWire/WirePlumber,
and reconnect the headset to determine whether a network tunnel is
triggering the problem.

For automatic discovery:

``` bash
mv ~/.config/pipewire/pipewire.conf.d/zeroconf-discover.conf \
   ~/.config/pipewire/pipewire.conf.d/zeroconf-discover.conf.disabled

systemctl --user restart pipewire pipewire-pulse wireplumber
```

For a persistent manual tunnel, disable its configuration similarly:

``` bash
mv ~/.config/pipewire/pipewire-pulse.conf.d/remote-sink.conf \
   ~/.config/pipewire/pipewire-pulse.conf.d/remote-sink.conf.disabled

systemctl --user restart pipewire pipewire-pulse wireplumber
```

------------------------------------------------------------------------

## 🐛 Known Issue: Bluetooth Headphones Stuck in Handsfree Mode

On **Fedora Linux 43 (Workstation Edition)** with **PipeWire 1.4.11**
and **WirePlumber 0.5.14**, enabling PipeWire network-audio tunnels can
cause Bluetooth headphones to lose their **A2DP / High Fidelity
Playback** profiles.

This was observed with **Bose QuietComfort 35**.

### Normal State

Normally, the Bluetooth card exposes A2DP and Handsfree profiles:

``` text
a2dp-sink-sbc
a2dp-sink-sbc_xq
a2dp-sink
headset-head-unit-cvsd
headset-head-unit
```

### Broken State

After enabling network audio, only the Handsfree profiles may remain:

``` text
headset-head-unit-cvsd
headset-head-unit
```

GNOME consequently shows only:

``` text
Handsfree - Bose QuietComfort 35
```

This results in low-quality HFP/mSBC audio instead of normal A2DP stereo
playback.

### What Triggers It

The issue was reproducible when using automatic discovery with:

``` text
libpipewire-module-zeroconf-discover
```

It was also observed when creating a remote sink manually with:

``` bash
pactl load-module module-tunnel-sink ...
```

Therefore, **Zeroconf itself does not appear to be the root cause**.
Zeroconf discovery automatically creates Pulse tunnel nodes, and the
problem appears to be associated with the presence or creation of these
network-audio tunnels.

### Workaround

Disable the network-audio configuration and restart the audio stack:

``` bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

After reconnecting the Bluetooth headphones, the A2DP profiles should
return.

For automatic Zeroconf discovery, disable the configuration with:

``` bash
mv ~/.config/pipewire/pipewire.conf.d/zeroconf-discover.conf \
   ~/.config/pipewire/pipewire.conf.d/zeroconf-discover.conf.disabled

systemctl --user restart pipewire pipewire-pulse wireplumber
```

For a persistent manual tunnel:

``` bash
mv ~/.config/pipewire/pipewire-pulse.conf.d/remote-sink.conf \
   ~/.config/pipewire/pipewire-pulse.conf.d/remote-sink.conf.disabled

systemctl --user restart pipewire pipewire-pulse wireplumber
```

### Diagnosis

Inspect the Bluetooth card:

``` bash
pactl list cards | sed -n '/Bose QuietComfort 35/,+45p'
```

When the bug is present, the `Profiles:` section contains only HSP/HFP
profiles instead of the expected `a2dp-sink-*` profiles.

### Tested Environment

``` text
Client:          Fedora Linux 43 (Workstation Edition)
PipeWire:        1.4.11
WirePlumber:     0.5.14
Headphones:      Bose QuietComfort 35
Remote receiver: Debian GNU/Linux 12 (Bookworm)
```

At the time of testing, this appears to be a **PipeWire/WirePlumber
Bluetooth profile interaction or regression**, rather than a Bluetooth
pairing problem or a problem with the headphones themselves.
