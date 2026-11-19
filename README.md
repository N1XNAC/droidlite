# droidlite

A deliberately stripped Android emulator. Runs **only what app testing needs**:
a bootable Android runtime where you can install an APK, use it, screenshot it,
read its logs, and run instrumentation tests. Everything else is turned off.

It uses the stock Google emulator binary (there is no practical way to
recompile it) plus a stock AOSP system image, then removes cost everywhere else:
the AVD hardware, the boot flags, the guest system apps, and two-thirds of the
emulator installation itself.

```
┌──────────────────────────────────────────────────────────────┐
│ KVM (hardware virtualization)  ← required, checked by doctor │
│  └─ Android x86_64 guest, 720p, 2 vCPU, 1.5–2 GB             │
│      └─ your APK + launcher + adb + logcat + screenshots     │
└──────────────────────────────────────────────────────────────┘
```

## Why not just use the normal emulator

A default Android Studio AVD asks for a modern phone: 1080×2400, 4 vCPU,
2–4 GB RAM, host GPU, Google Play services, camera, sensors, radios, SD card,
snapshots, and a 6–8 GB data partition. On a 4-core / 8 GB laptop that is the
difference between "usable" and "swap thrashing".

The biggest wins, in order:

| Change | Effect |
|---|---|
| `aosp_atd` or plain AOSP image instead of Play Store image | no GMS/Search/Play, ~1 GB less resident |
| 720×1280 @ 320 dpi instead of 1080×2400 | 40% fewer pixels to compose/upload |
| disable ~30 unused system apps | less boot work, less idle CPU/RAM |
| animation scales = 0 | launches feel instant, less GPU work |
| `-no-audio`, no camera/GPS/BT/NFC/sensors/sdcard | fewer guest HALs and threads |
| 1.5–2 GB guest RAM, 2 vCPU | fits alongside the host IDE |
| cold boot, no snapshots | no snapshot I/O, no stale state |
| `prune` unused QEMU binaries | frees disk, shorter install |

## Install / first run

```bash
git clone https://github.com/N1XNAC/droidlite.git
cd droidlite
./droidlite doctor        # KVM present? image already there?
./droidlite setup         # emulator + minimal image + AVD (one time, ~2 GB)
./droidlite start        # boots; first boot applies the trim automatically
./droidlite install ~/azx-assistant-debug.apk
./droidlite run com.azx.assistant
./droidlite shot         # PNG of the current screen
./droidlite logcat
```

Optional, so `droidlite` works from anywhere:

```bash
ln -sf ~/android-dev/droidlite/droidlite ~/.local/bin/droidlite
```

## Commands

| Command | What it does |
|---|---|
| `droidlite setup` | install `emulator` + `aosp_atd` image, create AVD, write trimmed config |
| `droidlite start` | boot (KVM, 2 vCPU, 720p, no audio/radios, trimmed on first boot) |
| `droidlite start --profile normal` | 4 vCPU / 3 GB / larger heap if you have headroom |
| `droidlite start --headless` | no emulator window; view with `droidlite screen` (scrcpy) |
| `droidlite start --image gapps` | API 35 Google APIs image, then same debloat |
| `droidlite stop` / `status` | shut down / show guest memory + host RSS |
| `droidlite install <apk>` | `adb install` + AOT compile for faster cold starts |
| `droidlite run <pkg>` | launch and wait until the first frame is drawn |
| `droidlite shot [file]` | screenshot straight off the framebuffer |
| `droidlite screen` | scrcpy at 1024 px / 30 fps / 2 Mbps (low-bandwidth mirroring) |
| `droidlite inspect <apk>` | package, SDK level, launch activity and ABIs before installing |
| `droidlite trim [--no-gms]` | re-apply debloat + settings; `--no-gms` also kills GMS |
| `droidlite untrim` | re-enable everything (nothing is deleted) |
| `droidlite reset` | wipe user data, keep image/AVD |
| `droidlite prune` | delete ARM/32-bit QEMU binaries from the SDK |
| `droidlite prune --headless` | also delete Qt + host Vulkan (≈430 MB more, no window ever) |
| `droidlite doctor` | KVM, image, AVD, disk/RAM checks |

## What exactly is trimmed

### AVD hardware (`config.ini`)

```
hw.cpu.ncore=2          hw.camera.back=none     hw.sensors.*=no (all of them)
hw.ramSize=1536         hw.camera.front=none    hw.bluetooth=no
hw.lcd=720x1280@320     hw.audioInput=no        hw.nfc=no
vm.heapSize=256         hw.audioOutput=no       hw.fingerprint=no
disk.data=2 GB          hw.gps=no               hw.sdCard=no
showDeviceFrame=no      PlayStore.enabled=false
```

Sensors are off on purpose: they cost HAL processes and wake-ups, and apps that
*need* an accelerometer (games) are not the target. Re-enable one key in
`~/.android/avd/dl_aosp_35.avd/config.ini` if you need it.

### Boot flags

`-no-boot-anim -no-audio -no-metrics -no-passive-gps -camera-back none
-camera-front none -netdelay none -netspeed full -partition-size 2048`
plus `-no-window` when headless.

### Guest system apps

`trim/debloat-aosp.txt` — Calendar, Clock, Gallery, Email, Music, AOSP browser,
Camera, Print, Dreams/live wallpapers, SIM toolkit, cell broadcast, NFC,
provisioning, Easter egg, traceur… ~30 packages.

`trim/debloat-google.txt` — only for `--image gapps`: Search, Maps, YouTube
(+Music), Photos, Gmail, Drive, Docs, Keep, Messages, Talkback, setup/sync
adapters, Play Games… GMS itself stays unless you pass `--no-gms`.

They are **disabled per user**, not deleted: `droidlite untrim` restores them,
and a `reset` brings back the pristine image.

### Emulator installation (`droidlite prune`)

The `emulator` package ships QEMU for every guest architecture plus ANGLE
libraries and a crash reporter. This project only ever runs x86_64 guests, so
the ARM/32-bit QEMU binaries are dead weight. Add `--headless` to also drop Qt
and the host Vulkan drivers, which are only used by the emulator window.

| | package size |
|---|---|
| stock install | ~863 MB |
| `prune` | ~670 MB |
| `prune --headless` | ~240 MB |

## Limits (read before filing a bug)

- **ABI.** x86_64 images run x86_64 native code. An APK containing only
  `arm64-v8a` libraries (like `azx-assistant-debug.apk`) will install but crash
  in `System.loadLibrary`. Check with:
  ```bash
  unzip -l app.apk | grep '^.*lib/'
  ```
  Fix at the source: add `x86_64` to `ndk { abiFilters }` (llama.cpp and
  onnxruntime both build for x86_64) and rebuild. `droidlite doctor` notes the
  ABI of the running image. There is no ARM translation in AOSP/Google APIs
  x86_64 images on modern API levels; an arm64 image would run under pure TCG
  emulation, which is too slow to be useful on this class of machine.
- **No Google Play / GMS** on the default image. Apps that hard-require Play
  Services need `--image gapps`.
- **KVM is mandatory.** Without `/dev/kvm` the emulator is unusably slow.
- Instrumentation tests work (`adb shell am instrument`); the AOSP ATD image
  (`--image atd`) is the fastest choice for CI-style runs but has a smaller
  system UI, so use `aosp` when you need to *look* at the app.

## Measured on this machine

| | Android Studio default AVD | droidlite (low) |
|---|---|---|
| guest RAM | 2048–4096 MB | 1536 MB |
| vCPU | 4 | 2 |
| screen | 1080×2400 @ 420 | 720×1280 @ 320 |
| system apps | ~120 | ~90 active (rest disabled) |
| emulator install | ~2.5 GB | ~0.67 GB after `prune` (0.24 GB with `--headless`) |

Boot time with KVM on a 4-thread laptop: printed by `droidlite start`; expect
15–30 s cold, roughly half of a default Play Store AVD.
