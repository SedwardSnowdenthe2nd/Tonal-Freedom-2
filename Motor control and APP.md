# Controlling a Tonal Without a Subscription — a Sideloaded Android Control App


*** Only do this after completing tonal10devie-unlock-guide.md

An unofficial, subscription-free control app for a **personally-owned** Tonal, built as a
sideloaded Android app that talks directly to the machine's motor-control board. This document
is the how-to: the architecture, the protocol, the build toolchain, the deploy flow, and — most
importantly — the gotchas that cost real time so you don't repeat them.

> This is not affiliated with, endorsed by, or supported by Tonal. It documents interoperability
> work on hardware the author owns. Nothing here is decompiled proprietary code; it describes a
> public (exported) Android service interface and the app built on top of it.

---

## ⚠️ Read this first — safety

This app commands a motor that applies real force through a cable to a handle or bar you hold.
A control failure can drop weight or fail to engage weight you are bracing against.

- **Never trust a control path in a position where a failure can injure you** (e.g. a braced
  barbell bench press) until you have personally proven it reliable across dozens of cycles **at
  a weight light enough to escape one-handed**.
- There is a known, unresolved control-ownership issue between the on-screen button and the
  handle's Bluetooth buttons (see [The unsolved problem](#the-unsolved-problem--control-ownership)).
  Until it's understood on your unit, **use one control exclusively for any lift where being
  pinned is possible.**
- Build and keep a physical escape plan for every position. Software is not a safety system.

Use entirely at your own risk.

---

## Why

Tonal's hardware is a capable digital-resistance trainer, but full functionality is gated behind
a subscription. If you own the machine and have local access to it, you can drive the resistance
board yourself and build whatever UI you want.

---

## What you need

- A Tonal you own, with developer/ADB access to its Android system (this writeup assumes you
  already have a shell on the device; obtaining that is out of scope here).
- A second machine on the same LAN to stage and push builds. This project used a **Raspberry Pi**
  reachable over SSH, with `adb` installed on the Pi and network access to the Tonal.
- A Linux build environment with the Android SDK command-line tools (`aidl`, `aapt2`, `d8`,
  `zipalign`, `apksigner`) and a JDK. No Android Studio / Gradle required.

Device characteristics observed on this unit (yours may differ):

- Pre-API-24 Android; `minSdk 21` works.
- **No root.** `adb root` prints *"restarting adbd as root"* but the shell stays `uid=2000`;
  there is no `su` binary anywhere. SELinux context `u:r:shell:s0`.
- Everything below is achievable **without root** — it relies on an exported service.

---

## How it works — the MCB service

The resistance board ("MCB") is driven through an **exported Android system service** that any
app on the device can bind to:

- Service: `com.riptlabs.mcb.service` → `com.riptlabs.common.mcb.RiptMcbService`
- Exported, **requires no permission**, runs as `uid 1000`.
- AIDL interface `IRiptMcbService`, with the key method:

  ```java
  void messageSend(String json);
  ```

You bind the service, register a listener to receive inbound messages, and send commands as JSON
strings. That's the entire control surface.

### Wire format

Every message is JSON:

```json
{ "msgtype": <int>, "body": { <snake_case fields> } }
```

### Message types

**Outbound (commands you send):**

| Name                  | msgtype | Purpose                                              |
|-----------------------|---------|------------------------------------------------------|
| `MASS_PROFILE_SET`    | 19      | Set the weight profile / engage weight               |
| `END_SET`             | 18      | End the current set                                  |
| `FEATURE_CONTROL_SET` | 4138    | Feature control; carries the e-stop trigger          |
| `BLE_BUTTON_EVENT`    | 8195    | Bluetooth accessory button event                     |

`FEATURE_CONTROL_SET` with feature type `TRIGGER_ESTOP = 13` fires a board-level emergency stop
(cuts the motors; latches until re-armed).

**Inbound (what the board streams back — identified by logging received `msgtype`s):**

| msgtype | Meaning                                              |
|---------|------------------------------------------------------|
| 4116    | Motor values / load telemetry (reported weight)      |
| 4120    | Temperature                                          |
| 4128    | Cart position                                        |
| 4129    | Lock position                                        |
| 4133    | Position / side                                      |
| 4141    | DFU / firmware                                       |
| 4147    | Comm stats                                           |
| 8192    | Accessory event (`accessory_id 166729` = the handle) |
| 8199    | Accessory battery                                    |

### The weight profile

`MASS_PROFILE_SET` carries the full profile: base weight, eccentric overload, chains, and a
"SmartFlex"-style modifier, plus a **location** selecting which motors/arms it applies to:

- `location` values seen: `ALL_INDEPENDENT = 3`, `VIRTUAL_DIFFERENTIAL = 2`.
- Weight is **per-arm**; a combined (barbell) number is `2 × per-arm`.

Exact field names live in the source (`McbMessages.java`); they're straightforward once you see
one profile go out.

---

## Building the APK without Gradle

The whole build is a shell pipeline:

```
aidl  →  aapt2  →  javac --release 8  →  d8  →  zipalign  →  apksigner
```

### Critical gotcha: `d8` crashes on lambdas / anonymous classes

Standalone `d8` (outside Gradle's desugaring pipeline) **crashes on anonymous inner classes and
lambdas.** This is the single biggest footgun in the project.

> **Rule: every listener and callback must be a named `static` nested class.** No lambdas, no
> anonymous `new OnClickListener(){…}`. This touches click handlers, service listeners,
> `Runnable`s, BLE callbacks — all of it.

### Other build-script gotchas

- **`unset JAVA_TOOL_OPTIONS`** before building, or injected options derail the toolchain.
- If your script pipes `javac` through `| … || true`, it will **mask compile errors.** Grep the
  build output for `error` explicitly and fail the build yourself if you find any.
- Use a **persistent debug keystore** so `adb install -r` (reinstall over existing) keeps working
  across builds. Generate it once and reuse it.
- Builds are non-deterministic (the APK md5 changes every build) — don't use md5 to detect
  "did it rebuild"; use it to detect "did the right file land on the device" (below).

---

## Deploying / sideloading

Build in your Linux environment, then stage through the Pi and install over ADB:

```bash
# from your build machine: copy the APK to the Pi
scp build/app-debug.apk squanto@<pi-ip>:~/TonalControlProto.apk

# on the Pi (adb reachable to the Tonal):
md5sum ~/TonalControlProto.apk          # verify this matches the build you intended
adb install -r ~/TonalControlProto.apk
adb shell am force-stop com.example.tonalproto
adb shell am start -n com.example.tonalproto/.MainActivity
```

### Gotcha: stale installs

The most confusing "nothing I changed took effect" failures were **stale APKs**, not code bugs.
Two causes:

1. Browsers/Windows save repeated downloads of the same name as `TonalControlProto(1).apk`,
   `(2)`, etc., so copying the fixed name pushes an **old** file.
2. Pushing before the new build actually finished.

**Fixes that end the confusion:**

- Put a **visible build tag in the app header** (e.g. `tonal · mcb · v27`). You can read the
  running version off the screen instantly.
- **`md5sum` the APK on the Pi** right before install and confirm it matches the build you meant
  to ship.

---

## Booting straight to your app

Goal: the machine powers on into your app instead of the stock launcher.

- **`pm disable-user <launcher>` is denied** without root (`SecurityException`, `uid=2000`).
- `adb root` does not actually elevate (see above), so you cannot disable the stock launcher.

**What works without root:** make your app a HOME app and set it as default.

1. Declare a HOME intent-filter in the manifest:

   ```xml
   <intent-filter>
     <action android:name="android.intent.action.MAIN" />
     <category android:name="android.intent.category.HOME" />
     <category android:name="android.intent.category.DEFAULT" />
   </intent-filter>
   ```

2. Open the system's home-selection screen and pick your app:

   ```bash
   adb shell am start -a android.settings.HOME_SETTINGS
   ```

The unit then boots to the app. (Declare `RECEIVE_BOOT_COMPLETED` + a boot receiver if you also
want to handle cold-boot explicitly.)

---

## Preventing silent OTA updates (optional, reversible)

A networked Tonal can pull OTA Android updates that may change or break the service surface you
depend on. Disabling the Tonal's WiFi stops that. It's reversible — turn WiFi back on when you
want updates. Only do this if you understand you're freezing the platform where it is.

---

## Features implemented (v27)

The reference app grew to include:

- **Weight engage / disengage** with a single dynamic toggle button (green ↔ red).
- **Barbell mode** — combined display up to 200 lb; the app sends half per arm, **hard-capped at
  100 lb/arm (200 combined)**. Chains/SmartFlex can never push a motor past the cap.
- **Eccentric overload, chains, and SmartFlex** modifiers, each with a safety clamp so the sum
  can't exceed the per-arm ceiling, and `(+lb)` breakdowns shown next to each.
- **Rest timer** with presets (15/30/45/60/90/120 s); controls hide and show a countdown with
  3-2-1 beeps.
- **Heart-rate monitor** over BLE (standard HRM GATT: service `0x180D`, characteristic `0x2A37`,
  CCCD `0x2902`), with on-screen pairing (tap to scan/select) and **HR zones 1–5** derived from
  an age → max-HR control.
- **Live graphs**: "REP POWER" and "REP RANGE", reset on engage.
- **Spotter / Burnout / Drop Set** modes.
- **Countdown volume** capped at 15% of real output (displayed as a friendly 0–100%).

---

## Hard-won lessons (the gotchas that cost time)

These are the non-obvious behaviors of the board. Internalize them before building your own UI.

1. **Do NOT re-send `MASS_PROFILE_SET` to an already-engaged board.** Re-asserting the profile
   makes the board **drop the weight.** An "engage watchdog" that re-fires the profile will
   actively defeat a successful engage. (Learned the hard way — it broke engagement entirely.)

2. **Do NOT send `END_SET` on disengage if you want on-screen re-engage to work.** `END_SET`
   closes the set and forces a hardware re-arm; afterward the on-screen engage won't apply weight
   until the set is re-armed. Disengage by sending weight-off at the matching location instead.

3. **Engage and disengage must use the same `location`.** If you engage at
   `VIRTUAL_DIFFERENTIAL (2)` but send weight-off at `ALL_INDEPENDENT (3)`, the weight won't drop.
   Match them.

4. **Bind the service once; remove the listener before re-adding it.** Duplicate binds stack
   multiple service listeners, so one button press is handled 2–3 times → random multi-toggling.
   Remove-before-add, and use `android:launchMode="singleTask"` so the activity isn't duplicated.

5. **The handle sends duplicate button events.** A single physical press can emit 2–3 events.
   A pure time-debounce isn't enough if you also have the stacked-listener bug (#4). Fix #4 first,
   then edge-trigger: track a `handleDown` boolean and act only on the PRESSED→(not pressed)
   transition.

6. **Synthetic `BLE_BUTTON_EVENT (8195)` messages are ignored by the board.** You cannot fake a
   handle press to grab control. (This matters for the ownership problem below.)

7. **Stale installs masquerade as code bugs.** Build stamp + md5 verify (see deploy section).

---

## The unsolved problem — control ownership

The one thing not solved by v27, and the reason for the safety warning.

The board hands **"ownership" of the next engage to whichever control last disengaged.** In
practice:

- Disengage with the handle → the **handle** owns the next engage; the on-screen engage won't
  apply weight.
- Disengage with the screen → the **screen/app** owns it; the handle engage won't apply weight.

Disengage (weight-off) generally works from either control; it's **engage** that's gated by
ownership. Attempts to override it from a sideloaded app all failed: faking the handle's button
event is ignored (#6 above), and a "claim ownership before engage" step didn't stick. This looks
like **firmware-level accessory arbitration**, below what a sideloaded app can reach.

**Working theory / open question:** the handle is likely a hardware peer with its own direct link
to the board, so pressing it reassigns ownership underneath the app regardless of what the app
does. The decisive test (not yet run): make the app *ignore* the handle button entirely, then
press it — if weight still engages/disengages, the handle has a private line to the board and the
two controls can't be unified from userspace. If pressing it does nothing, both controls route
through the app and a single shared engage/disengage action *can* be built.

**Practical mitigation today:** keep ownership on one control. For any lift where a failed
engage/disengage could pin you, use the **handle exclusively** (engage and disengage both), and
don't touch the screen mid-lift — ownership only shuffles when you mix controls.

---

## Project layout (reference)

```
app/src/main/java/com/example/tonalproto/
  MainActivity.java      # UI, service binding, engage/disengage, timers, HR, graphs
  McbMessages.java       # builds the JSON command messages (profile, weight-off, end-set, e-stop)
  HeartRate.java         # BLE HRM client + pairing
  GraphView.java         # rep power / rep range plots
app/src/main/res/layout/activity_main.xml
app/src/main/AndroidManifest.xml
build_apk.sh             # the aidl→aapt2→javac→d8→zipalign→apksigner pipeline
```

---

## Disclaimer

Unofficial and unaffiliated. For use on hardware you own. Driving a motorized resistance trainer
with custom software carries a real risk of injury and of damaging your equipment. The authors
provide no warranty and accept no liability. Verify every control path yourself, at safe weights,
before relying on it.
