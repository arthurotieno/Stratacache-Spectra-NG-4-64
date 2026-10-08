# Stratacache Spectra NG-4-64 → Linux Install

Field notes for repurposing a **Stratacache Spectra NG-4-64** digital signage player
as a general-purpose Linux box or browser kiosk.

These boxes turn up cheaply on the secondary market. They are **NVIDIA Tegra K1**
machines, electrically and software-wise very close to an NVIDIA Jetson TK1
development board. Many arrive with a half-finished factory image that boots to a
flickering screen and no login prompt, which makes them look bricked. They are not.

Everything below was verified on real hardware. Where something is unverified it is
marked **[unverified]**.

---

## 1. What the hardware actually is

| | |
|---|---|
| SoC | NVIDIA Tegra K1 (T124), chip marking **CD575M-A1** — 4× Cortex-A15 32-bit + Kepler GPU |
| RAM | 2 GiB |
| Storage | 58.2 GiB eMMC (`DF4064`) — only ~15 GB is partitioned from the factory |
| Board ID | reports as `jetson-tk1`, `NVIDIA Tegra124 PM375`, board_info `0x0177:0x0000:0x45:0x44:0x00` |
| Bootloader | **mainline U-Boot 2018.05** (`2018.05-gc50329da15`, built Oct 31 2019) |
| OS as shipped | NVIDIA L4T R21.8 — Ubuntu 14.04 armhf, kernel `3.10.40-ge16a41a05c9e` |
| Display | 2× HDMI via `tegradc.0` / `tegradc.1`; HDMI clock capped at 297 MHz |
| Network | Realtek RTL8168g/8111g gigabit over PCIe (`r8169`, appears as `eth0`) |
| Audio | HDA + Realtek RT5639 codec |
| Other | SATA controller, SD/MMC slot (`mmc 1`), USB host ports, RS-232 console, RTC (AS3722), hardware watchdog |

Notes:

- The **"64" in the model name is the eMMC size**, not the RAM.
- U-Boot identifies itself as a Jetson TK1 and the stock Jetson TK1 device tree
  (`tegra124-jetson_tk1-pm375-000-c00-00.dtb`) drives this board correctly.
- There is **no BIOS and no bootable-USB-installer path** like a PC. It boots
  U-Boot from eMMC, then uses U-Boot's `distro_bootcmd` / extlinux mechanism.

---

## 2. The fault in the factory image

Symptom: the box powers on, HDMI flickers endlessly, and there is never a login
prompt on screen or on serial.

Cause: **the root filesystem is owned by UID 1000 instead of root.** Whoever imaged
these boxes extracted NVIDIA's sample root filesystem without root privileges. A
leftover `/README.txt` saying *"Download and extract the sample filesystem to this
directory."* is the smoking gun.

Consequences:

- `/bin/sh`, `/usr/bin/sudo`, `/bin/mount`, `/usr/bin/passwd` and friends are owned
  by `ubuntu` while keeping their setuid bit — so running them *drops* you from root
  to UID 1000 instead of raising you to root.
- `login` and PAM cannot read `/etc/shadow` properly, so no one can log in.
- The flickering is a separate, cosmetic issue (see §9).

The fix is a `chown` pass plus restoring the setuid/setgid bits. Section 7.

---

## 3. Tools and downloads

### Hardware you need

- **USB-to-RS232 cable** for the box's DB9 console port. A genuine
  [Prolific PL2303GC](https://prolificusa.com/) works; so do FTDI and CP2102 cables.
  - A **null-modem (crossover) adapter** may be needed if nothing appears at any baud rate.
  - PL2303GC is USB ID `067b:23a3`. Its Linux driver support landed around kernel 5.9,
    so it will **not** work on an old Ubuntu 14.04/16.04 host. On Windows use
    Prolific's current driver.
- **USB flash drive**, FAT32, ideally USB 2.0 and small. 4 GB is plenty.
- **USB keyboard** and an **HDMI display**. A plain 1080p TV is better than a
  high-resolution monitor (see §9).
- Ethernet cable.

### Software (Windows host)

| Tool | Link | Why |
|---|---|---|
| Tera Term | https://teratermproject.github.io/ | Serial console. Has the transmit-delay setting you will need. |
| PuTTY | https://www.putty.org/ | Alternative serial/SSH client (no transmit delay, so less good for U-Boot). |
| Notepad++ | https://notepad-plus-plus.org/downloads/ | Required — you must save config files with **Unix (LF)** line endings. |
| 7-Zip | https://www.7-zip.org/ | Opens NVIDIA's `.tbz2` archives. |
| Advanced IP Scanner | https://www.advanced-ip-scanner.com/ | Finding the box's IP on your LAN. |

### NVIDIA L4T R21.8 (optional, for reference or a clean flash)

- Driver package: https://developer.nvidia.com/embedded/dlc/tk1-driver-package-r218
- Sample root filesystem: https://developer.nvidia.com/embedded/dlc/sample-root-filesystem-r218
- Release notes: https://developer.download.nvidia.com/embedded/L4T/r21_Release_v8.0/Tegra_Linux_Driver_Package_Release_Notes_R21.8.pdf
- Quick start guide (R21.5, still applicable): https://developer.download.nvidia.com/embedded/L4T/r21_Release_v5.0/l4t_quick_start_guide.txt

### Useful references

- Debian on Jetson TK1: https://wiki.debian.org/InstallingDebianOn/NVIDIA/Jetson-TK1
- `tegrarcm` (recovery-mode flashing tool): https://github.com/NVIDIA/tegrarcm
- Tegra recovery USB IDs discussion: https://forums.developer.nvidia.com/t/tk1-with-jetpack-3-1-nvflash-fails-when-usb-device-id-is-0955-7740/56896

---

## 4. Serial console

Settings: **115200 baud, 8 data bits, no parity, 1 stop bit, no flow control.**

In Tera Term:

1. *Serial*, pick the COM port (check Device Manager → Ports).
2. **Setup → Serial port**: 115200, 8, none, 1, flow control **none**.
3. **Setup → Serial port → Transmit delay: 50 msec/char, 100 msec/line.**
   This is not optional. U-Boot has no flow control and silently drops pasted
   characters; without a delay, commands arrive truncated and you will chase
   phantom errors for hours.
4. **File → Log...** to capture everything to a file.

If you see nothing: try 57600 and 9600, then try a null-modem adapter. Garbled
characters mean wrong baud rate; total silence means wiring.

---

## 5. Getting to the U-Boot prompt

Power-cycle the box and tap a key in the terminal immediately — the autoboot
countdown is about 2 seconds. You want:

```
Tegra124 (Jetson TK1) #
```

**Note:** U-Boot's prompt also ends in `#`, which is easy to confuse with a root
shell. The tell:

- `Tegra124 (Jetson TK1) #` → U-Boot
- bare `#` or `ubuntu@tegra-ubuntu:~$` → Linux

U-Boot only listens on the serial port (`In: serial`), so a USB keyboard cannot
interrupt the boot.

### Boot layout reference

```
boot_targets = mmc1 mmc0 usb0 pxe dhcp
mmc list     = sdhci@700b0400: 1   (SD/removable slot)
               sdhci@700b0600: 0   (eMMC)
```

**`mmc1` and `usb0` are tried before the internal eMMC**, which is why a bootable SD
card or USB stick can take over without writing anything to the box.

eMMC partition map (`part list mmc 0`) — standard L4T GPT:

| # | Name | Notes |
|---|---|---|
| 1 | APP | ext4 root filesystem (~15 GB), bootable |
| 2 | DTB | raw device tree (start LBA `0x01c91000`) |
| 3 | EFI | |
| 4 | USP | |
| 5–7 | TP1–TP3 | |
| 8 | WB0 | |
| 9 | UDA | |

Everything past partition 9 — roughly **42 GB** — is unallocated.

The stock system boots via `/boot/extlinux/extlinux.conf` on APP. The saved U-Boot
environment is blank ("bad CRC, using default environment"), so there is no custom
boot script to fight.

`Net: No ethernet found` in U-Boot is normal — the NIC sits behind PCIe and U-Boot
does not bring it up. Linux handles it fine.

---

## 6. Build the rescue USB stick

The goal is a stick that boots the box's own kernel straight to a root shell,
writing nothing to the eMMC.

### 6a. Copy the kernel and DTB off the eMMC

At the U-Boot prompt, with the stick inserted. **Type these by hand** or paste with
the transmit delay set:

```
usb reset
load mmc 0:1 0x81000000 /boot/zImage
```

Confirm it prints `6230800 bytes read`, then:

```
fatwrite usb 0:1 0x81000000 zImage 0x5F1310
load mmc 0:1 0x81000000 /boot/tegra124-jetson_tk1-pm375-000-c00-00.dtb
fatwrite usb 0:1 0x81000000 board.dtb 0xEA3E
fatls usb 0:1
```

`fatls` must show `zImage` at **6230800** and `board.dtb` at **59966**. Sizes are
given explicitly in hex because `${filesize}` is stale from whatever was loaded
last — the single most time-wasting trap in this whole process.

Alternative: pull `Linux_for_Tegra/kernel/zImage` out of the R21.8 driver package
with 7-Zip and copy it to the stick from Windows.

### 6b. Add the boot config

On the stick, create `extlinux/extlinux.conf`. **Open it in Notepad++ and set
Edit → EOL Conversion → Unix (LF) before saving.** Windows CRLF makes U-Boot read
the kernel path as `/zImage\r` and fail with "Unable to read file /zImage".

```
TIMEOUT 10
DEFAULT rescue

MENU TITLE USB rescue

LABEL rescue
      MENU LABEL rescue shell
      LINUX /zImage
      FDT /board.dtb
      APPEND console=ttyS0,115200n8 no_console_suspend=1 lp0_vec=2064@0xf46ff000 mem=2015M@2048M memtype=255 ddr_die=2048M@2048M section=256M pmuboard=0x0177:0x0000:0x02:0x43:0x00 tsec=32M@3913M otf_key=c75e5bb91eb3bd947560357b64422f85 usbcore.old_scheme_first=1 core_edp_mv=1150 core_edp_ma=4000 tegraid=40.1.1.0.0 debug_uartport=lsport,3 power_supply=Adapter audio_codec=rt5640 modem_id=0 fbcon=map:1 commchip_id=0 usb_port_owner_info=0 lane_owner_info=6 emc_max_dvfs=0 touch_id=0@0 board_info=0x0177:0x0000:0x02:0x43:0x00 net.ifnames=0 root=/dev/mmcblk0p1 rw rootwait tegraboot=sdmmc gpt init=/bin/sh
```

That is the factory command line with `console=tty1` removed (so the shell lands on
serial) and `init=/bin/sh` added.

> `otf_key` and `pmuboard`/`board_info` values came off one specific unit. They are
> almost certainly identical across these boxes, but if a unit misbehaves, read its
> own `extlinux.conf` and copy its values. **[unverified across units]**

### 6c. Boot it

```
usb reset
run bootcmd_usb0
```

### 6d. Manual boot, if the menu route misbehaves

All in one session — a power cycle wipes RAM, so loads and `bootz` must not be
separated by a reset:

```
usb reset
load usb 0:1 0x81000000 zImage
load usb 0:1 0x82000000 board.dtb
setenv bootargs console=ttyS0,115200n8
setenv bootargs ${bootargs} root=/dev/mmcblk0p1 rw rootwait
setenv bootargs ${bootargs} init=/bin/sh
setenv bootargs ${bootargs} mem=2015M@2048M
setenv bootargs ${bootargs} lp0_vec=2064@0xf46ff000
setenv bootargs ${bootargs} tegraid=40.1.1.0.0
setenv bootargs ${bootargs} debug_uartport=lsport,3
printenv bootargs
bootz 0x81000000 - 0x82000000
```

Always check `printenv bootargs` before `bootz`. **Total silence after
"Starting kernel ..." means the arguments were truncated** — it is essentially never
a hardware problem.

You should reach a `#` prompt within ~20 seconds. `/bin/sh: 0: can't access tty; job
control turned off` is expected and harmless.

---

## 7. Repair the root filesystem

At the rescue `#` prompt. `/` is already read-write (it is in `bootargs`).

First confirm the fault:

```sh
ls -l /bin/sh /usr/bin/sudo
```

Owned by `ubuntu` → broken, continue. Owned by `root` with `-rwsr-xr-x` on sudo →
this unit is fine, skip to §8.

### 7a. Fix ownership

```sh
chown -R 0:0 /bin /boot /etc /lib /media /mnt /opt /root /run /sbin /srv /usr /var /tmp /home
chown -R 1000:1000 /home/ubuntu
```

Takes a few minutes on eMMC.

### 7b. Restore setuid and setgid bits

**This step is mandatory.** `chown` clears setuid/setgid bits, so skipping it leaves
you with a system that logs in but has no working `sudo`, `mount` or `passwd`.

```sh
chmod u+s /bin/su /bin/mount /bin/umount /bin/ping /bin/ping6 /bin/fusermount
chmod u+s /usr/bin/passwd /usr/bin/sudo /usr/bin/chsh /usr/bin/chfn /usr/bin/gpasswd /usr/bin/newgrp
chmod u+s /usr/lib/openssh/ssh-keysign /usr/lib/dbus-1.0/dbus-daemon-launch-helper /usr/lib/policykit-1/polkit-agent-helper-1
chgrp shadow /etc/shadow /etc/gshadow /sbin/unix_chkpwd /usr/bin/expiry
chmod 640 /etc/shadow /etc/gshadow
chmod g+s /sbin/unix_chkpwd /usr/bin/expiry
chgrp tty /usr/bin/wall /usr/bin/write
chmod g+s /usr/bin/wall /usr/bin/write
```

Errors about paths that don't exist on this image are harmless.

### 7c. Set a password and reboot

```sh
mount -t proc proc /proc
passwd ubuntu
ls -l /usr/bin/sudo /usr/bin/passwd /bin/mount
sync
reboot -f
```

The three binaries must read `-rwsr-xr-x 1 root root`. `passwd` must say
**"password updated successfully"**.

### 7d. Optional: back up the eMMC first

Worth doing before any of this, from the rescue shell with a second USB drive
mounted:

```sh
dd if=/dev/mmcblk0 of=/mnt/usb/emmc-full.img bs=4M
dd if=/dev/mmcblk0boot0 of=/mnt/usb/boot0.img
dd if=/dev/mmcblk0boot1 of=/mnt/usb/boot1.img
```

`boot0`/`boot1` hold the bootloader and memory configuration — the parts you cannot
reconstruct.

---

## 8. First normal boot

Pull the USB stick, power on, and let it boot. You should get `tegra-ubuntu login:`
on the HDMI display. Log in as `ubuntu`, then:

```bash
export TERM=linux
echo 'export TERM=linux' >> ~/.bashrc
sudo -i                 # must give a root prompt
cat /etc/nv_tegra_release
ip addr
ping -c3 8.8.8.8
```

There is no SSH server or serial login prompt on the factory image, so the HDMI
console plus USB keyboard is your way in until you install one:

```bash
sudo apt-get install openssh-server
```

---

## 9. Display: fixing the flicker

The flickering on some monitors is HDMI mode negotiation, not a fault. The driver
picks the display's highest mode; on a 24-inch 1920×1200-class monitor that came out
as a **279 MHz pixel clock**, right at the Tegra K1's ~297 MHz ceiling, so the link
dropped and renegotiated forever:

```
hdmi_state_machine: Reset → Check Plug → Check EDID → Enabled → Reset → ...
```

On a 1080p TV it settled at 148.5 MHz and stayed in "Enabled".

Options, easiest first:

1. **Use a 1080p display.** Confirmed fix.
2. Swap the HDMI cable — marginal cables fail exactly this way at high clocks.
3. Pin the mode in X (see §10) with an explicit `Modes "1920x1080"`.

---

## 10. Kiosk setup

Goal: boot straight into a full-screen browser, no desktop environment.

### Prerequisite — package sources

Ubuntu 14.04 reached end of life in 2019, so `ports.ubuntu.com` no longer carries it.
EOL releases move to `old-releases.ubuntu.com`, and the ARM archive lives under
`ubuntu-ports` there. Replace `/etc/apt/sources.list` with:

```
deb http://old-releases.ubuntu.com/ubuntu-ports/ trusty main universe
deb http://old-releases.ubuntu.com/ubuntu-ports/ trusty-updates main universe
```

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo nano /etc/apt/sources.list      # replace contents with the two lines above
sudo apt-get update
```

**[unverified]** — this is the correct location by Ubuntu's own convention for EOL
ports releases, but it could not be confirmed against the live server. If
`apt-get update` returns 404s, try `ports.ubuntu.com/ubuntu-ports` with the same
suite names before concluding the archive is gone, and if both fail go to §10d.

Expect `Release file expired` warnings whichever host works — the archive is years
old. Suppress them with:

```bash
sudo apt-get -o Acquire::Check-Valid-Until=false update
```

### 10a. Minimal X plus a browser

```bash
sudo apt-get update
sudo apt-get install --no-install-recommends \
    xserver-xorg xinit x11-xserver-utils \
    matchbox-window-manager unclutter
sudo apt-get install chromium-browser
```

If `chromium-browser` isn't available, `firefox` from the same archive works; both
will be roughly 2019-vintage. **[unverified which version the archive still has]**

L4T installs its own `/etc/X11/xorg.conf` using the `tegra` driver. Leave it alone if
it exists — that is what gives you accelerated output. To pin the resolution, add to
the `Monitor`/`Screen` section:

```
SubSection "Display"
    Modes "1920x1080"
EndSubSection
```

### 10b. The kiosk session

`~/.xinitrc`:

```sh
#!/bin/sh
xset s off
xset -dpms
xset s noblank
unclutter -idle 1 &
matchbox-window-manager -use_titlebar no &
exec chromium-browser \
    --kiosk \
    --noerrdialogs \
    --disable-infobars \
    --disable-session-crashed-bubble \
    --disable-translate \
    --check-for-update-interval=31536000 \
    --window-size=1920,1080 \
    http://your-url-here
```

```bash
chmod +x ~/.xinitrc
```

### 10c. Autologin and autostart

14.04 uses **upstart**, not systemd. Edit `/etc/init/tty1.conf` and change the exec
line to:

```
exec /sbin/getty -8 38400 -a ubuntu tty1
```

Then append to `~/.bash_profile` (create it if absent):

```sh
if [ -z "$DISPLAY" ] && [ "$(tty)" = "/dev/tty1" ]; then
    startx
fi
```

Reboot. The box should come up straight into the browser.

Two things to be aware of:

- The **hardware watchdog** (`tegra_wdt`) is enabled at probe. If the kiosk wedges
  the box may reset itself — usually helpful, occasionally confusing.
- With 2 GB of RAM and a 2014-era browser, keep the page light. Avoid a big swap file
  on eMMC; `zram-config` is kinder to the flash. **[unverified on this image]**

### 10d. If there is no working package archive

Options, in order of sanity:

1. Download armhf `.deb` files on another machine, carry them over on the USB stick,
   and `sudo dpkg -i *.deb`. Dependency resolution is manual and tedious.
2. Use a lighter browser that needs fewer dependencies — `links2 -g` or
   `netsurf-fb` run directly on the framebuffer with no X at all. Both are very
   limited on modern sites.
3. **Install Debian into the free 42 GB instead** (see §11). Debian armhf is still
   supported, so you get working package management and a current browser. This is
   more work up front but is the route that doesn't dead-end.

---

## 11. Further work: Debian in the free space

Not done yet, but the groundwork is all in place:

- ~42 GB of the eMMC is unpartitioned past partition 9.
- U-Boot is mainline and already boots via extlinux, so a second root filesystem with
  its own `/boot/extlinux/extlinux.conf` can be added as another menu entry, leaving
  the factory system intact as a fallback.
- Tegra124 is well supported by mainline Linux, including the GPU via `nouveau`
  (GK20A), so a current Debian armhf with a 6.x kernel should give accelerated
  graphics and a current Firefox. **[unverified]**
- `debootstrap` from another Linux machine is the usual way to build the rootfs.

Alternatively, a full clean flash with L4T R21.8 via NVIDIA's `flash.sh` is possible
but was *not* needed and carries real risk: `flash.sh` writes the Jetson TK1 BCT
(memory timings) to your board. Note also that in recovery mode these boxes enumerate
as USB ID **`0955:7740`**, not the `0955:7140` that NVIDIA's `nvflash` expects —
the low byte `0x40` is what identifies Tegra124, and the open-source `tegrarcm`
matches only that byte, so it works where `nvflash` refuses.

---

## 12. Gotchas, collected

Every one of these cost real time:

1. **Pasting into U-Boot drops characters.** Set a transmit delay or type by hand.
   Truncated commands produce misleading errors (`File not found /boot/zge`) and
   silent kernel boots.
2. **`usb start` may find nothing where `usb reset` works.** Use `usb reset`.
3. **`${filesize}` is stale** after any failed `load`. Pass sizes explicitly to
   `fatwrite` or you will write a 746-byte "kernel".
4. **CRLF breaks `extlinux.conf`.** Symptom: "Unable to read file /zImage" plus
   "Ignoring unknown command:" lines. Save as Unix LF.
5. **`chown` clears setuid bits.** Always follow a `chown -R` with the `chmod u+s`
   list, or you trade one broken system for another.
6. **U-Boot's prompt and a root shell both end in `#`.** Check which one you're
   talking to before typing Linux commands.
7. **No serial login prompt** on the factory image — the serial console is
   output-only once Linux is up. U-Boot is where serial input works.
8. **`load` and `bootz` must be in the same power cycle.** RAM is cleared on reset.
9. The DTB in the raw `DTB` partition is ~72 KB and is **not** the same as
   `/boot/tegra124-jetson_tk1-pm375-000-c00-00.dtb` (59966 bytes). Use the file.

---

## 13. Appendix: factory boot configuration

`/boot/extlinux/extlinux.conf` as shipped:

```
TIMEOUT 30
DEFAULT primary

MENU TITLE Jetson-TK1 eMMC boot options

LABEL primary
      MENU LABEL primary kernel
      LINUX /boot/zImage
      FDT /boot/tegra124-jetson_tk1-pm375-000-c00-00.dtb
      APPEND console=ttyS0,115200n8 console=tty1 no_console_suspend=1 lp0_vec=2064@0xf46ff000 mem=2015M@2048M memtype=255 ddr_die=2048M@2048M section=256M pmuboard=0x0177:0x0000:0x02:0x43:0x00 tsec=32M@3913M otf_key=c75e5bb91eb3bd947560357b64422f85 usbcore.old_scheme_first=1 core_edp_mv=1150 core_edp_ma=4000 tegraid=40.1.1.0.0 debug_uartport=lsport,3 power_supply=Adapter audio_codec=rt5640 modem_id=0 android.kerneltype=normal fbcon=map:1 commchip_id=0 usb_port_owner_info=0 lane_owner_info=6 emc_max_dvfs=0 touch_id=0@0 board_info=0x0177:0x0000:0x02:0x43:0x00 net.ifnames=0 root=/dev/mmcblk0p1 rw rootwait tegraboot=sdmmc gpt
```

Useful file sizes for verification:

| File | Bytes | Hex |
|---|---|---|
| `/boot/zImage` | 6230800 | `0x5F1310` |
| `tegra124-jetson_tk1-pm375-000-c00-00.dtb` | 59966 | `0xEA3E` |
| `/boot/extlinux/extlinux.conf` | 827 | `0x33B` |

---

## License

Documentation released into the public domain (CC0). No affiliation with Stratacache
or NVIDIA. Modifying these devices is at your own risk.
