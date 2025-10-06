# 🐧 Kali Nethunter Chroot on Android (Rooted Termux Guide)
© 2025 - Written by Kagenou

---
## ⛓️‍💥 In XFCE4 Desktop
![XFCE4 Desktop](./images/example1.png)

## 🫆 In SSH Kali
![SSH Kali](./images/example2.png)

## 📖 Overview
This guide explains **how to install and run Kali Nethunter (full rootfs) with XFCE4 desktop** inside **Termux using chroot** on a **rooted Android device**.  
You’ll get a full Kali Linux environment — including GUI and audio — with `Termux:X11` and `PulseAudio`.

> ⚠️ **Disclaimer:** This process requires root access. Proceed carefully — you are fully responsible for any damage to your device.

---

## ⚙️ Requirements
- Android device **with root access**
- **Termux** (latest version from [F-Droid](https://f-droid.org/en/packages/com.termux/))
- **Termux-x11** (latest from [termux-x11 releases](https://github.com/termux/termux-x11/releases))
- **Internet connection** to download Kali rootfs

---

## 🧩 Step 1: Update & Install Required Packages

Open Termux and run:

```bash
pkg update -y && pkg upgrade -y
```
```bash
pkg install tur-repo root-repo x11-repo -y
```
```bash
pkg install neofetch busybox tsu wget termux-x11 pulseaudio -y
```

---

## 🧰 Step 2: Fix tsu Binary Path (if it fails to run)

If `tsu` doesn't detect the su binary, fix its search paths:

```bash
sed -i 's|SU_BINARY_SEARCH=.*|SU_BINARY_SEARCH=("/system/xbin/su" "/system/bin/su" "/debug_ramdisk/su")|' $PREFIX/bin/tsu
```

---

## 📦 Step 3: Download Kali Nethunter RootFS (ARM64)

```bash
wget https://kali.download/nethunter-images/current/rootfs/kali-nethunter-rootfs-full-arm64.tar.xz
```

---

## 📂 Step 4: Extract Kali Filesystem

Run as root in Termux:

```bash
tsu
```
```bash
tar xvf kali-nethunter-rootfs-full-arm64.tar.xz -C ..
```
Exit root back to normal Termux user:
```bash
exit
```

---

## 🚀 Step 5: Create Launch Script

Create a new script `startkali.sh`:

```bash
cat <<'EOF' > startkali.sh
#!/data/data/com.termux/files/usr/bin/bash
if [ "$(whoami)" = "root" ]; then
  echo -e "
ROOT IS NOT ALLOWED HERE!
"
  exit 1
fi
BBX="$PREFIX/bin/busybox"
KALIROOT="/data/data/com.termux/files/kali-arm64"
TMPDIR="$KALIROOT/tmp"
XDG_RUNTIME_DIR="$KALIROOT/tmp/XDG"

# Start PulseAudio (for sound)
pulseaudio --start --exit-idle-time=-1 --system=false   --load="module-native-protocol-tcp auth-ip-acl=127.0.0.1 auth-anonymous=1" >/dev/null 2>&1 &

# Bind mounts (requires tsu/su)
su -c "$BBX mount -o remount,dev,suid /data"
su -c "$BBX mount --bind /dev $KALIROOT/dev"
su -c "$BBX mount --bind /dev/pts $KALIROOT/dev/pts"
su -c "$BBX mount --bind /proc $KALIROOT/proc"
su -c "$BBX mount --bind /sys $KALIROOT/sys"

# Start X server (Termux:X11) and enter chroot
termux-x11 :0 -ac >/dev/null 2>&1 &
X11_PID=$!
su -c "$BBX chroot $KALIROOT /bin/su - root"

# Cleanup
kill -9 $X11_PID 2>/dev/null || true
pulseaudio --kill
su -c "umount -lf $KALIROOT/dev/pts" 2>/dev/null || true
su -c "umount -lf $KALIROOT/dev" 2>/dev/null || true
su -c "umount -lf $KALIROOT/proc" 2>/dev/null || true
su -c "umount -lf $KALIROOT/sys" 2>/dev/null || true

echo -e "
© Kagenou - 2025"
EOF

chmod +x startkali.sh
```

Run it:

```bash
sh startkali.sh
```

---

## 🌐 Step 6: Network Configuration (Inside Kali)

Once inside the chroot (as root), configure DNS and helpers:

```bash
mv /etc/resolv.conf /etc/resolv.conf.ori || true
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "127.0.0.1 localhost" > /etc/hosts
groupadd -g 3003 aid_inet || true
groupadd -g 3004 aid_net_raw || true
groupadd -g 1003 aid_graphics || true
usermod -g 3003 -G 3003,3004 -a _apt || true
usermod -G 3003 -a root || true
```

---

## 🕒 Step 7: Create Login Scripts & XDG dir

```bash
mkdir -p /tmp/XDG
chown kali:kali /tmp/XDG || true
cd /var/lib || true
cat <<'EOF' > login
export DISPLAY=:0
export TMPDIR=/tmp
export XDG_RUNTIME_DIR=/tmp/XDG
export XKB_CONFIG_ROOT=/usr/share/X11/xkb
EOF

cat <<'EOF' > logout
pkill -9 -U "$(id -u)"
EOF

chmod a+x login logout
echo "source /var/lib/login" >> /home/kali/.bashrc
echo "source /var/lib/logout" >> /home/kali/.bash_logout
source /var/lib/login || true
```

Update apt:
```bash
sudo apt update -y && sudo apt upgrade -y
```

---

## 🧩 Step 8: Fix PAM / SSH Issues

If you get `Could not open session` or PAM errors, remove the pam_keyinit line:

```bash
sed -i '/pam_keyinit.so/d' /etc/pam.d/sshd || true
sed -i '/pam_keyinit.so/d' /etc/pam.d/su-l || true
```

Give sudo NOPASSWD for kali user (optional, risky):

```bash
echo "kali ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers
```

---

## 👤 Step 9: Make Start Alias & Switch User

Change start script to default into `kali` user instead of `root`:

```bash
sed -i 's|/bin/su - root|/bin/su - kali|' startkali.sh
echo 'alias kali="~/startkali.sh"' >> ~/.bashrc
source ~/.bashrc
chmod +x ~/startkali.sh
```

Start Kali as regular user:

```bash
kali
```

---

## 🔊 Step 10: Enable Sound System

In Termux host (outside chroot), add Pulse server env:

```bash
echo 'export PULSE_SERVER=127.0.0.1' >> ~/.bashrc
echo 'export PULSE_SERVER=127.0.0.1' >> ~/.zshrc
source ~/.bashrc || true
```

---

## 🖥️ Step 11: Create Desktop Launcher (desk)

Create a simple launcher to start XFCE4 inside chroot:

```bash
sudo mkdir -p /usr/local/bin
sudo tee /usr/local/bin/desk > /dev/null <<'EOF'
#!/bin/bash
set -euo pipefail
echo "Starting XFCE4 desktop..."
eval $(dbus-launch --sh-syntax --exit-with-session)
xfce4-session >/tmp/xfce4.log 2>&1 &
XFCE_PID=$!
echo "XFCE4 running in background. Switch to Termux:X11 to view the desktop."
(
  while pgrep -f "xfce4-session" >/dev/null 2>&1; do
    sleep 2
  done
  pkill -9 -P 1 -u kali 2>/dev/null || true
  pkill -9 -U "$(id -u)" 2>/dev/null || true
) & disown
EOF

sudo chmod +x /usr/local/bin/desk
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.zshrc
```

---

## ✅ DONE!
🎉 You now have **Kali Nethunter (full rootfs) with XFCE4** running inside Termux using **chroot**!

**Username:** `kali`  
**Password:** `123`  
**To start GUI:** `desk`

---

## Troubleshooting & Notes
-This guide only work on arm64
- If `termux-x11` fails to show display, check that the Termux:X11 app has permission and is running on your Android device.
- If `XDG_RUNTIME_DIR` ownership mismatch appears (`uid 100000` vs `uid 1000`), create a dedicated XDG folder inside the chroot and chown it appropriately.
- For audio issues, ensure pulseaudio is running in Termux host and `PULSE_SERVER` is set to `127.0.0.1`.
- Always backup important data before experimenting with chroot or rooting.

---
