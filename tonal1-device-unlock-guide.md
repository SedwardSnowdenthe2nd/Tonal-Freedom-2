# Unlocking a Tonal 1 You Own — Device-Owner / Kiosk Removal

A field-tested procedure for removing the Android **Device Owner** that locks a Tonal 1
into its kiosk, so you can control the Android tablet you own — install a launcher, use
adb, run your own apps.

---

## ⚠️ READ THIS FIRST — Scope, Safety, Legal

**What this guide does:** removes the Android Device Owner (`com.tonal.tonallauncher`)
that pins the machine into its kiosk launcher. After this, the Tonal is a normal,
unlocked Android tablet you control.

**What this guide deliberately does NOT do:**
- It does **not** touch the motor / force / resistance / safety control.
- It does **not** bypass the Tonal subscription.

Both were left completely alone, on purpose. This is a *device-unlock* guide, nothing more.

**🛑 SAFETY — the important part.** The Tonal's motors can produce **hundreds of pounds
of force** on a cable attached to your body. The spotter, eccentric, chains, drop-set,
burnout and SmartFlex behaviors are **safety-critical**. Do **not** use the access this
guide gives you to run custom or third-party motor-control code (e.g. driving the motors
directly) during a workout. A bug in hobby motor code — a failed-rep that doesn't shed
load, an eccentric that loads heavier than you can arrest — can seriously injure you.
If you ever experiment with the motor hardware at all, keep people clear of the mechanism,
have a physical emergency disconnect, and never test alone. The validated force/safety
system is Tonal's firmware; leave it that way. This guide stops at the device lock for a
reason.

**⚖️ LEGAL.** This is about controlling hardware you own outright (a right-to-repair
position). Circumventing software access controls can implicate laws such as the US DMCA
§1201; the rules are genuinely unsettled and vary by jurisdiction. This is not legal
advice, and you are responsible for your own use.

**💥 RISK.** Flashing firmware can brick a device if interrupted. **Back up first** (step 2
is not optional). Everything here is at your own risk.

---

## Target hardware

- **Tonal 1** (not Tonal 2 — unverified/unsupported here).
- SBC: InnoComm "Hercules" board, MediaTek SoC, **Android 6.0** (build `MRA58K`).
- The Android board exposes a micro-USB port.

## Tools

- A Linux host (a Raspberry Pi 4 works; any Linux box will do).
- A good USB **data** cable.
- [`mtkclient`](https://github.com/bkerler/mtkclient) — MediaTek BROM flashing tool.
- `magiskboot` (shipped inside the Magisk APK) for unpacking/repacking the boot image.

---

## Why the obvious approaches fail (and what actually works)

The kiosk is enforced by an Android **Device Owner**. Its state lives in
`/data/system/device_owner.xml`.

- You **can't** remove it from Settings (the entry is greyed out by design).
- You **can't** remove it with `dpm` (device owners are protected).
- **Magisk root does not help** on this board: Magisk's daemon never initializes
  correctly here, so `su` returns a *hollow* root — `id` shows `uid=0`, but
  `CapEff: 0000000000000000` (zero Linux capabilities). It can't delete the protected
  file, can't write `/data/adb`, can't `setenforce`.

**The method that works:** modify the **boot image** so that `init` itself deletes the
device-owner file on every boot, *before* `system_server` starts. `init` runs with full
capabilities very early, so it succeeds where restricted root fails — and deleting before
`system_server` reads it means the owner never loads.

> Note / lesson: on this board, `init` **can** unlink `/data/system/device_owner.xml`
> under *enforcing* SELinux — no permissive mode needed. We also confirmed the MediaTek
> bootloader **ignores the boot image's kernel cmdline**, so trying to force
> `androidboot.selinux=permissive` via the cmdline does nothing. Don't waste time on it.

---

## Procedure

### 1. Set up mtkclient
Install mtkclient on the host per its README. A udev rule (or running as root) lets it
talk to the MediaTek BROM over USB. Confirm it can catch the device in BROM by powering
the Tonal fully off, starting an mtk command, then cold-booting it.

### 2. Back up the boot partition — DO NOT SKIP
```bash
# with the device in BROM:
python mtk.py r boot boot.bin          # read the boot partition
# (a full backup of all partitions is even better; keep copies in TWO places)
```
Keep `boot.bin` safe. It is your one-flash recovery image. If anything goes wrong later:
```bash
python mtk.py w boot boot.bin          # restores a known-good, bootable device
```
The BROM lives in ROM and can't be erased, so a bad/half-written boot is always
recoverable this way.

### 3. Get magiskboot and unpack the boot image
`magiskboot` is inside the Magisk APK as a static arm64 binary:
```bash
mkdir work && cd work
cp ../boot.bin .
unzip -q -o -j /path/to/Magisk-v27.0.apk "lib/arm64-v8a/libmagiskboot.so" -d .
mv -f libmagiskboot.so magiskboot && chmod +x magiskboot
./magiskboot unpack boot.bin           # -> kernel, ramdisk.cpio, ...
```

### 4. Insert the delete into `init.rc`
Extract the init script and add the deletes *before every framework start*:
```bash
./magiskboot cpio ramdisk.cpio "extract init.rc init.rc"
```
The framework (zygote → system_server) is started by `class_start main`, which appears in
a few encryption-gated blocks (`on nonencrypted`, `trigger_restart_min_framework`,
`trigger_restart_framework`). Inserting the delete before **each** one guarantees it runs
after `/data` is mounted and before `system_server` reads the owner:
```bash
awk '/^[[:space:]]*class_start main[[:space:]]*$/{
       print "    rm /data/system/device_owner.xml";
       print "    rm /data/system/device_policies.xml"
     } {print}' init.rc > init.rc.new && mv init.rc.new init.rc
```
> `rm` is a valid `init` built-in on this Android 6 build (the stock `init.rc` already
> uses it), so no helper script is needed.

### 5. (Optional but practical) Enable adb so you can verify and set a launcher
The stock image ships adb off and USB in MTP mode. To get an adb shell for the next steps,
edit `default.prop`:
```bash
./magiskboot cpio ramdisk.cpio "extract default.prop default.prop"
sed -i 's/^ro\.adb\.secure=1/ro.adb.secure=0/;
        s/^ro\.secure=1/ro.secure=0/;
        s/^ro\.debuggable=0/ro.debuggable=1/;
        s/^persist\.sys\.usb\.config=mtp/persist.sys.usb.config=adb/' default.prop
# and force the adb USB function up after boot (defeats the launcher's mtp override):
printf '\non property:sys.boot_completed=1\n    setprop sys.usb.config adb\n' >> init.rc
```
> **Security note:** `ro.secure=0` / `ro.debuggable=1` is a real security downgrade. See
> *Hardening* below — rebuild a clean boot (just the step-4 delete, none of these flags)
> once you've got your launcher set.

### 6. Repack and flash
```bash
./magiskboot cpio ramdisk.cpio "add 0600 default.prop default.prop"   # if you did step 5
./magiskboot cpio ramdisk.cpio "add 0750 init.rc init.rc"
./magiskboot repack boot.bin boot-new.img
# flash via BROM:
python mtk.py w boot boot-new.img      # watch for 100%; if a write dies, re-run (don't boot)
```

### 7. Boot and confirm the owner is gone
```bash
adb shell dumpsys device_policy
```
Success looks like an **"Enabled Device Admins"** section with the Tonal launcher listed as
a plain admin, and **no "Device Owner:" line**. The owner is removed.

### 8. Set your own home launcher
Owner removal alone doesn't change your home screen — the Tonal launcher is still set as
the default home, so the machine still *lands* there. Install a launcher and pick it:
```bash
adb install your-launcher.apk
adb shell am start -a android.settings.HOME_SETTINGS   # opens the Home-app picker
```
On the Tonal screen, select your launcher. (The stock AOSP `com.android.launcher3` is often
already present.) You now boot into a normal Android home; the Tonal app is just an icon.

---

## Gotchas & lessons learned

- **Magisk gives hollow root on this board** (`CapEff=0`). Don't go down the Magisk path;
  the init-level delete is the move.
- **The MTK bootloader ignores the boot.img cmdline** — forcing SELinux permissive that way
  does nothing. You don't need permissive anyway; `init` can unlink under enforcing here.
- **The owner lives in memory too.** Deleting the file *after* `system_server` has read it
  won't take effect until a clean restart — which is exactly why we delete it in `init`,
  before the framework starts.
- **adb may be blocked by the owner** before removal (debugging restriction). Enabling it in
  `default.prop` as in step 5 gets you in.
- **USB reliability:** on a Pi, the USB controller can wedge after hours of flashing — writes
  start failing earlier each attempt. `sudo reboot` the Pi to reset its USB stack before
  flashing, and don't yank cables at the host mid-session.
- **Recovery is always available:** keep `boot.bin`; a half-written boot is fixed by
  re-flashing it over BROM.

## Hardening (recommended once you're done)
If you enabled adb in step 5, rebuild a boot with **only** the step-4 delete (none of the
`ro.secure=0` / `ro.debuggable=1` flags) and flash that, to restore normal device security
once your launcher is set.

---

## What was intentionally left out, and why

- **Motor / force / safety control — untouched.** This is the validated system that keeps
  you safe under load. Do not replace it with custom or third-party control code for live
  workouts. Serious-injury risk. This is the single most important line in this document.
- **Subscription — not bypassed.** If features that were marketed and sold with your machine
  are now behind a subscription, the constructive route is consumer-protection pressure
  (documented complaints to the vendor, your state consumer-protection office, or the FTC),
  not cracking the paywall.

*Shared in the spirit of owning what you bought. Be careful, back up, and keep people clear
of the machine.*
