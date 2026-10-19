# GT-I6410 (Vodafone 360 M1) - Root Shell via Widget Installer

Persistent root shell on the Samsung GT-I6410 (Vodafone 360 "M1", LiMo platform)
without flashing, without opening the phone, and fully reversible.
Tested on PDA **I6410BUIK5** / CSC VODIJD. Should apply to other LiMo firmware
revisions of the M1 as long as the widget installer accepts unsigned packages.

The phone's Linux userspace (browser included) already runs everything as
UID 0 - the only missing piece was a write primitive. The stock widget
installer extracts ZIP entries without sanitizing `../` paths, which allows
writing arbitrary files into `/opt` as root. This package uses that to place
a boot script that Samsung's own `startApps.sh` prefers over its default.

**Result:** after reboot the phone exposes a CDC-Ethernet interface over USB
with a root telnet shell on port 23.

## Files

| File | Purpose |
|------|---------|
| `fgroot4.wgt` | The exploit package (installs the boot hook) |
| `restore_fg.wgt` | Rollback: neutralizes the hook, restores stock behavior |

## Requirements

- Samsung GT-I6410 (Vodafone 360 M1), LiMo firmware
- USB cable
- A Linux PC (for `ip` and `telnet`; other OSes work with equivalents)

## Important note

**This is a potentially dangerous modification. Read this before installing.**

- While this modification is installed, **anyone with a USB cable gets a
  passwordless root shell** on the phone whenever it is booted. The shell has
  no authentication and full system access.
- Tested on exactly one device (I6410BUIK5, CSC VODIJD). Other firmware
  revisions may behave differently - the exploit relies on Samsung's
  `/opt/etc/startAppsFg.sh` override hook being present in your firmware.
- Nothing outside the writable `/opt` partition is ever touched, and the
  original boot script stays in place on the read-only system partition as a
  fallback. The worst *known* failure mode is the foreground-app startup
  being skipped after an error, leaving the phone UI incomplete, recoverable
  by installing `restore_fg.wgt` or reflashing stock firmware. Unknown risks
  cannot be ruled out; treat this as potentially bricking your device.
- **Note on recovery-by-reflash:** the stock firmware is **not included** in
  this release (I have one, but its licensing is unclear to me. Feel free to contact me if you're interested regarding this). At the time of writing, the only available firmware online is on a paid website.
- Removing the modification (`restore_fg.wgt`) also removes the remote shell.
- **Only install this on a device you own.** I do not take responsibility for
  any damage this may cause to your device, or for anything you do with the
  shell.

## Install

### Step 1 - Allow unsigned app installation

Open the phone dialer and enter:

```
*#35767#
```

In the menu that appears, set installation to **Allowed**, then back out.

### Step 2 - Connect the phone in file-transfer mode

Connect the phone via USB. When the USB-mode selection appears, pick the
file-transfer mode (**MTP / Mass storage** - anything that lets you copy
files onto the phone storage works).

### Step 3 - Copy the packages

Copy both files to the phone storage (e.g. into *Other files*):

- `fgroot4.wgt`
- `restore_fg.wgt` (keep this around for later rollback)

### Step 4 - Install the package on the phone

Disconnect USB, open the phone's **file browser**, navigate to where you
copied the files, and tap `fgroot4.wgt`. Confirm the installation. The
widget ("FgRoot4") appears in your app list - opening it once is optional,
it only verifies the install.

### Step 5 - Reboot with USB connected

Power the phone **off**, connect the USB cable, power it **on**. After boot
the USB-mode dialog appears again - pick anything (e.g. MTP). It doesn't
matter: the exploit already bound the USB gadget to a network interface.

Note: If you select MTP (transfer files to computer) right after boot the screen will show "Connecting...", if this bothers you, either reconnect the phone or use another option. MTP will not be available as long as fgroot is installed. To reverse this use restore_fg.wgt

### Step 6 - Give your PC an address

A new network interface appears (usually named `enx…` or `enp…u1…`):

```bash
ip a    # find the new interface
sudo ip addr add 192.168.77.1/24 dev <interface>
sudo ip link set <interface> up
```

### Step 7 - Connect

```bash
telnet 192.168.77.2
```

Press **Enter** once. You are dropped into a BusyBox `ash` as root:

```
# id
uid=0(root) gid=0(root)
```

The shell survives reboots (the hook is a boot script) and re-attaches
whenever the phone is booted with USB connected.

## Rollback / Uninstall

1. Install `restore_fg.wgt` on the phone exactly like in step 4 (file
   browser → tap → confirm). It replaces the boot hook with a neutral
   shim that just launches Samsung's original script.
2. Reboot. The phone now boots 100% stock again (no network gadget, no
   telnet).

No partition is ever flashed or modified outside `/opt`, and the original
boot script remains untouched on the read-only system partition as a
fallback at all times.

## Troubleshooting

| Symptom | Check |
|---|---|
| No network interface on PC | Re-plug USB; run `dmesg | tail` - you should see a CDC-Ethernet device register |
| Interface there, but ping fails | Did you assign `192.168.77.1/24`? (`ip a`) |
| Ping works, telnet refused | The telnet daemon is spawned by `inetd`; wait ~10 s (a watchdog loop restarts it), then retry `telnet 192.168.77.2 23` |
| Widget rejected as "Invalid app" | Step 1 (`*#35767#` → Allowed) was not active during install |
| Anything else | Open the phone browser and navigate to `file:///opt/etc/esc_boot.log` (tap the link from a local HTML page - typed file URLs get mangled by the address bar). The log shows the boot script's output. |

## How it works (short version)

1. `*#35767#` enables installing unsigned widgets (JIL-format packages).
2. The stock widget installer runs as root and follows `../` inside ZIP
   entry names, but confines writes to `/opt` - enough, because:
3. Samsung's `/usr/etc/startApps.sh` explicitly prefers
   `/opt/etc/startAppsFg.sh` over its read-only default when present - a
   designed-in override point, executed as root at every normal boot.
4. The override script swaps the USB gadget from serial (`g_serial`) to
   Ethernet (`g_ether`), configures `usb0` to `192.168.77.2/24`, starts a
   private `inetd` instance with its own config that serves a root telnet
   shell on port 23 (BusyBox 1.11's `telnetd` is inetd-only), keeps both
   alive with a small watchdog loop, and finally `exec`s Samsung's original
   script so the phone boots normally.
5. File permissions are delivered via the ZIP `external_attr` field, which
   the installer applies with `chmod`.

## Credits / License

Reverse-engineered on a privately owned device. Released for device preservation and homebrew purposes -
do whatever you want with it, attribution appreciated.
