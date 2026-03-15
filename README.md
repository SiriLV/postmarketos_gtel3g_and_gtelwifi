# PostmarketOS on Samsung Galaxy Tab E 9.6 (SM-T561 / SM-T560)

Universal guide for setting up PostmarketOS on Samsung Galaxy Tab E tablet.

> [!NOTE]
> **Blue screen on boot:** If you see a blue screen with a "broken" Samsung logo — this is normal. The system is booting, SSH access is available.

## 📱 Hardware Status

| Feature | Status | Note |
| :--- | :---: | :--- |
| **Screen / Touchscreen** | ✅ | Requires calibration (see below) |
| **WiFi** | ✅ | |
| **USB (OTG/ADB)** | ✅ | Keyboard required for initial setup |
| **Audio** | ⚠️ | Works after patch (T561 only?) |
| **Battery** | ✅ | |
| **Hardware buttons** | ✅ | |
| **Sleep (Suspend)** | ✅ | |
| **Bluetooth** | ❌/❓ | Needs testing |
| **Camera** | ❌/❓ | |
| **GPS / 3G** | ❌ | |

---

## 🚀 Getting Started (SSH)

By default, device access is via USB network.

**Login credentials:**
*   **IP:** `172.16.42.1`
*   **User:** `user`
*   **Password:** `1`

```bash
ssh user@172.16.42.1
```

### Connecting to WiFi
```bash
doas nmcli device wifi list
doas nmcli device wifi connect "SSID_NAME" password "YOUR_PASSWORD"
doas apk update && doas apk upgrade
```

---

## 🖥️ Recommended Desktop Environments

The choice of environment affects performance and usability:

1. **LXQt** — for those who are lazy to configure. Lightweight, works out of the box
2. **i3wm** — for those who want the gold standard. Tiling manager, maximum performance
3. **MATE** — a good option, balance of simplicity and functionality
4. **XFCE4** — works, but requires startup fix + no application display
5. **Openbox** — minimalist window manager
6. **none** — for advanced users, manual configuration

---

## 🖥️ Setting up Graphical Environment (MATE / XFCE)

By default, the graphical shell may not start.

### 1. Installing and configuring LightDM (for MATE / XFCE)

> **Note:** LightDM is required for autostart of the graphical shell in MATE and XFCE.

```bash
doas apk add lightdm lightdm-gtk-greeter
doas rc-update add lightdm default
```

Edit the config `/etc/lightdm/lightdm.conf`.
Make changes in the `[LightDM]` and `[Seat:*]` sections:

```ini
[LightDM]
logind-check-graphical=false

[Seat:*]
user-session=xfce
autologin-user=user
autologin-user-timeout=0
```

> For MATE replace `user-session=xfce` with `user-session=mate`

Restart the service:
```bash
doas service lightdm restart
```

### 2. Screen and Touchscreen Rotation
Create directory for Xorg configs:
```bash
doas mkdir -p /etc/X11/xorg.conf.d
```

**File `/etc/X11/xorg.conf.d/10-monitor.conf` (Screen rotation):**
```xorg
Section "Device"
    Identifier "LCD"
    Driver "fbdev"
    Option "Rotate" "CW"
EndSection
```

**File `/etc/X11/xorg.conf.d/99-calibration.conf` (Touch calibration):**
```xorg
Section "InputClass"
    Identifier "calibration"
    MatchProduct "sec_touchscreen"
    Option "TransformationMatrix" "0 1 0 -1 0 1 0 0 1"
EndSection
```

Apply changes with reboot:
```bash
doas reboot
```

---

## 🔊 Audio Setup

### Installing and configuring audio system

**Step 1. Installing packages and adding to groups:**
```bash
doas addgroup user audio
doas addgroup user video
doas addgroup user input
doas apk add pulseaudio alsa-plugins-pulse pavucontrol alsa-utils
```

**Step 2. PulseAudio Configuration**

Overwrite `/etc/pulse/daemon.conf`, and paste this code:
```ini
enable-shm = no
enable-memfd = no
exit-idle-time = -1
resample-method = trivial
flat-volumes = no
allow-module-loading = yes
```

Also file `/etc/pulse/default.pa`:
```ini
# Protocols
load-module module-native-protocol-unix auth-anonymous=1 socket=/run/user/10000/pulse/native
# load-module module-dbus-protocol # Disabled to avoid errors

# Restore
load-module module-device-restore
load-module module-stream-restore
load-module module-card-restore
load-module module-augment-properties
load-module module-switch-on-port-available

# LOAD SPREADTRUM DRIVER (by name)
load-module module-alsa-sink device=hw:sprdphone tsched=0 mmap=0 sink_name=sprd_out
load-module module-alsa-source device=hw:sprdphone tsched=0 mmap=0 source_name=sprd_in

# Defaults
set-default-sink sprd_out
set-default-source sprd_in

# Additional
load-module module-always-sink
load-module module-position-event-sounds
load-module module-role-cork
```

And file `/etc/pulse/client.conf`:
```ini
enable-shm = no
enable-memfd = no
autospawn = yes
```

**Step 4. Restart**

```bash
doas killall -9 pulseaudio 2>/dev/null
rm -rf /run/user/10000/pulse/*
pulseaudio -D
```

**Step 5. Find card number**
You need to know the card number (-c X) to apply mixer settings.
Enter:

```bash
aplay -l | grep sprdphone
```
The number after the word card is your number.
If it shows card 0 -> use -c 0 in commands
If it shows card 2 -> use -c 2 in commands

Further in the instructions I use -c 2, as this is my latest version. If the card changes, change the number.

**Step 6. Mixer Setup (Amixer)**
This is the complete sequence for enabling the entire path for Spreadtrum SC7730/8830. Execute in blocks.

Block 1: Enable power "switches"
```bash
amixer -c 2 sset 'Speaker Function' on
amixer -c 2 sset 'Speaker2 Function' on
amixer -c 2 sset 'VBC DACL DG' on
amixer -c 2 sset 'VBC DACR DG' on
```

Block 2: Routing (Connect the wires). Without this, sound is lost inside the chip.
```bash
amixer -c 2 sset 'SPKL Mixer DACLSPKL' on
amixer -c 2 sset 'SPKR Mixer DACRSPKR' on
amixer -c 2 sset 'VBC DA IIS Mux' 'sprd-codec'
amixer -c 2 sset 'VBC' 'ap'
```

Block 3: Volume
```bash
amixer -c 2 sset 'SPKL' 100%
amixer -c 2 sset 'SPKR' 100%
amixer -c 2 sset 'DACL' 100%
amixer -c 2 sset 'DACR' 100%
```

Block 4: Mute - The trickiest moment
Here we try two options.

Option A (Standard):
```bash
amixer -c 2 sset 'Speaker Mute' off
amixer -c 2 sset 'Speaker2 Mute' off
#Test sound with command
paplay /usr/share/sounds/alsa/Front_Center.wav
```

Option B (If there's still no sound):
```bash
amixer -c 2 sset 'Speaker Mute' on
amixer -c 2 sset 'Speaker2 Mute' on
#Test sound with command
paplay /usr/share/sounds/alsa/Front_Center.wav
```

**Step 7. Saving**
After sound is working, be sure to save the settings:
```bash
doas alsactl store
doas rc-update add alsa default
```

> **Important!** Without saving, all settings will reset after reboot.

---

## 💾 Mounting microSD Card and USB Drives

### Manual mounting via terminal

**Step 1. Identify disk name**

Insert memory card or USB drive and run:
```bash
lsblk
```

> If command not found: `doas apk add util-linux`

**How to distinguish devices on Galaxy Tab E:**
- `mmcblk0` — internal tablet memory (don't touch)
- `mmcblk1p1` — usually SD card
- `sda1` (or `sdb1`) — USB flash drive via OTG

**Step 2. Create mount point**
```bash
doas mkdir -p /mnt/usb
```

**Step 3. Mount device**

For USB drive:
```bash
doas mount /dev/sda1 /mnt/usb
```

For SD card:
```bash
doas mount /dev/mmcblk1p1 /mnt/usb
```

Now files are accessible in `/mnt/usb`

**Step 4. Safe removal**

Before disconnecting device, run:
```bash
doas umount /mnt/usb
```

### exFAT and NTFS Support

To work with Windows formats, install drivers:
```bash
doas apk add exfat-fuse ntfs-3g
```

For exFAT you can use:
```bash
doas mount.exfat-fuse /dev/sda1 /mnt/usb
```

### Automatic mounting on boot

**Step 1. Find device UUID:**
```bash
doas blkid
```

Find the line for your device (e.g., `/dev/mmcblk1p1`) and copy the UUID.

**Step 2. Edit `/etc/fstab`:**
```bash
doas nano /etc/fstab
```
Add a line (replace UUID and filesystem type):
```
UUID=YOUR-UUID-HERE  /mnt/usb  exfat  defaults,lazytime,noatime,uid=1000,gid=1000,fmask=0022,dmask=0022,nofail,x-systemd.device-timeout=5  0  0
```

Examples for different filesystems:
- **exFAT:** `exfat  defaults,lazytime,noatime,uid=1000,gid=1000,fmask=0022,dmask=0022,nofail,x-systemd.device-timeout=5  0  0`
- **NTFS:** `ntfs-3g  defaults,lazytime,noatime,uid=1000,gid=1000,fmask=0022,dmask=0022,nofail,x-systemd.device-timeout=5  0  0`
- **ext4:** `ext4  defaults,lazytime,noatime,uid=1000,gid=1000,fmask=0022,dmask=0022,nofail,x-systemd.device-timeout=5  0  0`
- **FAT32:** `vfat  defaults,lazytime,noatime,uid=1000,gid=1000,fmask=0022,dmask=0022,nofail,x-systemd.device-timeout=5  0  0`

> The `nofail` parameter prevents boot errors if the card is not inserted.

**Step 3. Check mounting:**
```bash
doas mount -a
```

If there are no errors, the device will mount automatically on every boot.

---

## 🛠️ Useful Software

### Recommended Applications

Installing essential tools:
```bash
doas apk add firefox-esr btop neofetch fish micro ranger qterminal
```

**Description:**
- **firefox-esr** — browser (stable Firefox version)
- **btop** — system resource monitoring
- **neofetch** — nice system information display
- **fish** — convenient command line shell
- **micro** — simple text editor for coding
- **ranger** — terminal file manager
- **qterminal** — lightweight terminal emulator

### Switching to Fish shell (optional)

For more comfortable terminal work:
```bash
chsh -s /usr/bin/fish
```

> After this, reboot.
