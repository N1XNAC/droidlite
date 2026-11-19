# droidlite

**A lightweight Android emulator for low-end PCs — run an APK without Android Studio.**

droidlite wraps the stock Android emulator and strips it down to the one job that
matters during development: install an app, use it, read its logs, take a
screenshot. It boots a 720p Android 15 device with **2 vCPUs and 1.5 GB of RAM**,
no Google Play, no camera, no sensors, no audio, no Bluetooth/NFC/GPS, and up to
35 built-in apps disabled. On a 4-core / 8 GB laptop that is the difference
between a usable test device and swap thrashing.

```text
stock AVD   1080×2400 · 4 vCPU · 2–4 GB · Play services · sensors · camera · audio
droidlite    720×1280 · 2 vCPU · 1.5 GB · AOSP         · none     · none  · none
```

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
![Platform: Linux](https://img.shields.io/badge/platform-Linux%20%7C%20macOS%20%7C%20WSL2-lightgrey)
![Guest RAM: 1.5 GB](https://img.shields.io/badge/guest%20RAM-1.5%20GB-brightgreen)
![Play Services: no](https://img.shields.io/badge/Google%20Play-not%20required-red)

## Quick start (60 seconds)

```bash
git clone https://github.com/N1XNAC/droidlite.git
cd droidlite
./droidlite doctor                 # checks KVM, images, RAM, disk
./droidlite setup                  # emulator + minimal AOSP image + AVD (~2 GB, once)
./droidlite start                  # boot; first boot also debloats the guest
./droidlite install app-debug.apk
./droidlite run com.example.app
./droidlite shot                   # screenshot of the running app
./droidlite logcat                 # live log stream
./droidlite stop
```

Optional, to call it from anywhere:

```bash
ln -sf "$PWD/droidlite" ~/.local/bin/droidlite
```

## What is droidlite?

droidlite is a **minimal Android emulator setup for app testing on low-spec
hardware**. It is not a fork of AOSP and it is not a new hypervisor: it uses
Google's emulator binary with KVM acceleration, then removes every feature that
costs CPU, RAM, boot time or disk but does not help you test an app.

The result is a normal ADB device. Anything that works with `adb` works with
droidlite: Gradle's `installDebug`, Android Studio's device picker, `scrcpy`,
`am instrument`, Espresso/UiAutomator, screenshots, logcat, `run-as`, and so on.

### Who it is for

- Developers on **4–8 GB RAM laptops** who cannot afford a full Play Store AVD.
- CI runners and containers that need a **headless Android device**.
- Anyone who wants to **run an APK without installing Android Studio**.
- AI coding agents that need a fast, disposable Android target.

### Who it is not for

- Games or apps that need an accelerometer, gyroscope, camera, GPS or Bluetooth.
- Anything that hard-depends on Google Play Services (use `--image gapps`).
- Production QA on many device profiles — use a device farm.

## Requirements

| Requirement | Notes |
|---|---|
| Linux x86_64 | Primary target. macOS and WSL2 work with the same emulator. |
| KVM | `/dev/kvm` must exist and be writable (Hypervisor.framework on macOS). Without acceleration the emulator is unusably slow. |
| 4 CPU threads | 2 are given to the guest. |
| 4 GB free RAM | The guest takes 1.5 GB (2 GB with `--ram 2048`). |
| ~8 GB disk | Emulator ~0.7 GB + image ~1.5 GB + user data 2 GB. |
| JDK 17 | Only for `sdkmanager`/`avdmanager`; `droidlite doctor` tells you if it is missing. |

## Commands

| Command | What it does |
|---|---|
| `droidlite doctor` | Verify KVM, image, AVD, RAM and disk before you start |
| `droidlite setup` | Install `emulator` + `default` AOSP image, create the AVD |
| `droidlite start` | Boot with KVM: 2 vCPU, 720p, no audio/radios/sensors, cold boot |
| `droidlite start --headless` | No window; view with `droidlite screen` (scrcpy) |
| `droidlite start --profile normal` | 4 vCPU / 3 GB if the host has headroom |
| `droidlite start --image gapps` | Google APIs image (adds ARM64 translation) |
| `droidlite stop` / `status` | Shut down / show guest memory and host RSS |
| `droidlite install <apk>` | `adb install` + AOT compile for faster cold starts |
| `droidlite inspect <apk>` | Package name, SDK level, launch activity, ABIs |
| `droidlite run <pkg> [activity]` | Launch and wait until the first frame is drawn |
| `droidlite logcat` / `logcat -d` / `logcat -c` | Stream / dump / clear logs |
| `droidlite shot [file.png]` | Screenshot without opening a window |
| `droidlite screen` | Interactive scrcpy at 1024 px / 30 fps / 2 Mbps |
| `droidlite shell`, `push`, `pull` | ADB passthrough |
| `droidlite trim` / `untrim` | Re-apply / revert the debloat list |
| `droidlite reset` | Wipe user data, keep the image and AVD |
| `droidlite prune` | Delete ARM/32-bit QEMU binaries (~184 MB) |
| `droidlite prune --headless` | Also delete Qt + host Vulkan (~430 MB more) |
| `droidlite list` | Installed images and AVDs |

All commands are non-interactive and safe to script. State lives in `.state/`
(active serial, pid, logs) inside the repo.

## How it compares

| | Android Studio AVD | droidlite | Waydroid / redroid | Genymotion |
|---|---|---|---|---|
| Runs on 4–8 GB RAM laptop | painful | **yes** | yes | yes |
| Needs kernel modules (binder) | no | **no** | yes | no |
| KVM required | yes | yes | no | yes |
| Full Play Store | optional | no (opt-in GApps) | no | paid |
| Headless / CI friendly | awkward | **yes** | yes | paid |
| Screen / density control | yes | **yes** (720p default) | partial | yes |
| Setup | IDE + ~8 GB | **2 commands, ~2 GB** | Docker + kernel work | installer |
| Real phone UI shell | yes | **yes** (AOSP launcher) | yes | yes |

droidlite is the pragmatic choice when the host is weak, but you still want a
real Android userspace that behaves like a phone for your app.

## What exactly gets trimmed

### Guest hardware

```ini
hw.cpu.ncore=2            hw.camera.back=none       hw.sensors.*=no
hw.ramSize=1536           hw.camera.front=none      hw.bluetooth=no
hw.lcd=720x1280 @ 320dpi  hw.audioInput=no          hw.nfc=no
vm.heapSize=256           hw.audioOutput=no         hw.fingerprint=no
disk.data=2 GB            hw.gps=no                 hw.sdCard=no
showDeviceFrame=no        PlayStore.enabled=false
```

That is 40 % fewer pixels to compose, two thirds of the vCPUs, half the RAM and
no sensor/radio HALs compared with a default Pixel AVD.

### Boot flags

`-no-boot-anim` `-no-audio` `-no-metrics` `-no-passive-gps`
`-camera-back none` `-camera-front none` `-no-snapshot` `-netdelay none`
`-netspeed full` `-partition-size 2048`, plus `-no-window` when headless.

### Up to ~35 guest system apps

Calendar, Clock, Gallery, Email, Music, AOSP browser, Camera, Print spooler,
live-wallpaper/dream services, SIM toolkit, cell broadcast, NFC, enterprise
provisioning, Easter egg, traceur and friends — listed in
[`trim/debloat-aosp.txt`](trim/debloat-aosp.txt). Google images additionally get
Search, Maps, YouTube, Photos, Gmail, Drive, Docs, Keep, Messages, Talkback and
setup/sync adapters from [`trim/debloat-google.txt`](trim/debloat-google.txt).

They are **disabled per user, never deleted**: `droidlite untrim` brings them
back, `droidlite reset` restores a pristine image.

### Emulator installation

`droidlite prune` deletes QEMU builds for guest architectures that can never run
here (ARM, ARM64, i386), ANGLE and the crash reporter. `--headless` also removes
Qt and host Vulkan. Sizes on this machine: **863 MB → 679 MB → 247 MB**.

## Performance

Measured on a 4-thread Intel Kaby Lake laptop with 8 GB RAM, API 35
(`--image gapps`), 2 vCPU, 2 GB guest RAM, headless + SwiftShader:

| Measurement | Result |
|---|---|
| First cold boot ever | ~151 s |
| Subsequent cold boot | **~59 s** |
| Guest RAM in use after boot | ~1.1 GB of 2.0 GB |
| Host RSS (QEMU, 2 GB guest) | ~3.0 GB |
| Install + AOT compile, 86 MB APK | ~21 s |
| Cold app launch to first frame | ~8 s |

Windowed mode (`--gpu host`) uses the Intel iGPU and boots noticeably faster
than software rendering. `droidlite start` prints the boot time on every run.

Verified end to end: an **arm64-only APK** (llama.cpp + ONNX Runtime native
libraries) installs and launches on the Google APIs image through NDK
translation; `libonnxruntime.so` loads from `lib/arm64-v8a` via
`libndk_translation.so`.

## AI agents

droidlite is designed to be driven by an LLM/agent without a human in the loop:

- Every command is non-interactive, has a stable name and exits non-zero on failure.
- Machine-readable summary: `./droidlite status --json`.
- Deterministic target: one AVD per image/API, fixed serial `emulator-5554`.
- State and logs are plain files in `.state/`.
- [`AGENTS.md`](AGENTS.md) contains the exact recipe, guardrails and failure table.
- [`llms.txt`](llms.txt) is a compact, crawler-friendly summary of this project.

Canonical agent loop:

```bash
./droidlite doctor
./droidlite start --headless
./droidlite inspect app-debug.apk
./droidlite install app-debug.apk
./droidlite run com.example.app
./droidlite shot /tmp/droidlite-screen.png
./droidlite logcat -d > /tmp/droidlite-log.txt
./droidlite stop
```

## FAQ

**Is droidlite faster than the Android Studio emulator?**
Yes on constrained hardware. It uses the same emulator and KVM but boots with a
720p framebuffer, two vCPUs, no Google services, no sensors and no snapshot I/O.
The guest has far less work to do at boot and while idle.

**How much RAM does it need?**
1.5 GB for the guest by default (`--ram 2048` if your app is heavy). The host
needs roughly another 0.5–1 GB for QEMU, so 4 GB free is comfortable.

**Does it need KVM?**
Yes. Without `/dev/kvm` QEMU falls back to software emulation and Android becomes
unusably slow. Run `./droidlite doctor` to check.

**Can I run it without a desktop / over SSH?**
Yes: `./droidlite start --headless` and `./droidlite screen` or `shot`. This is
also the recommended mode for CI.

**Can it run ARM-only APKs?**
AOSP images cannot. The `--image gapps` Google APIs image includes ARM64 NDK
translation (`libndk_translation.so`), so ARM-only APKs install and run there,
slower. For real speed, rebuild with `x86_64` in `ndk.abiFilters`.
`./droidlite inspect app.apk` tells you which ABIs an APK has.

**Does it support Google Play Services?**
Not on the default image. Start with `--image gapps`; droidlite still debloats
the heavy Google apps. Add `trim --no-gms` if you also want GMS gone.

**Can I use it from Android Studio or Gradle?**
Yes. It is a normal ADB device. Run `droidlite start`, then deploy from the IDE
or `./gradlew installDebug`.

**Is the emulator binary modified?**
No. droidlite only changes configuration, system apps and which bundled QEMU
binaries are kept. That keeps it compatible with every emulator update.

**Does it work on Windows or macOS?**
The CLI is Bash; use WSL2 on Windows or the built-in Bash on macOS. The emulator
itself is Google's and supports all three hosts.

**How do I get the default emulator back?**
`./droidlite untrim` re-enables system apps; `./droidlite reset` wipes user data.
Nothing outside the repo is modified except the SDK packages and the AVD.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `emulator exited during boot` | `tail -50 .state/emulator.log`; usually KVM permission or missing image |
| `INSTALL_FAILED_NO_MATCHING_ABIS` | APK has no x86_64 libs — see the ARM FAQ above |
| App installs but crashes in `System.loadLibrary` | same ABI issue; `./droidlite inspect app.apk` |
| Boot is extremely slow | check `/dev/kvm`; use `--gpu host` instead of headless SwiftShader |
| Adb says `unauthorized` | `adb kill-server && ./droidlite start` |
| Screen is tiny/huge | edit `hw.lcd.width/height/density` in the AVD `config.ini` |

## Limits

- x86_64 guests only; no ARM system images (pure software emulation is too slow).
- No Play Store, no Google account, no SafetyNet/Play Integrity.
- Sensors, camera, GPS, Bluetooth, NFC and audio are deliberately absent.
- One emulator at a time (fixed port 5554) — enough for app testing, not a farm.

## License

MIT. See [LICENSE](LICENSE).
