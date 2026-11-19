# AGENTS.md — using droidlite from an AI agent

droidlite is a **headless-capable, low-resource Android emulator** for testing
APKs. This file is the contract: commands, state, guardrails, and how to recover.
Humans can read it too.

## TL;DR

```bash
./droidlite doctor                  # preflight, exit 1 if unusable
./droidlite start --headless        # boot 720p Android, waits for boot_completed
./droidlite install app-debug.apk   # adb install + AOT compile
./droidlite run com.example.app     # launch, waits for first frame
./droidlite shot /tmp/screen.png    # screenshot
./droidlite logcat -d > /tmp/log.txt
./droidlite stop
```

Do not open a GUI. Do not use Android Studio. Everything above is
non-interactive and exits non-zero on failure.

## Preconditions

1. `./droidlite doctor` must exit 0. It checks `/dev/kvm`, the emulator binary,
   the system image and host resources.
2. First use on a machine needs `./droidlite setup` (downloads ~2 GB once).
3. The CLI is Bash. It needs `adb`, which ships with the Android SDK it manages.

## State and outputs

| Path | Meaning |
|---|---|
| `.state/serial` | Active ADB serial, normally `emulator-5554` |
| `.state/pid` | Emulator process |
| `.state/flavor` | Active image flavour (`aosp`, `atd`, `gapps`) |
| `.state/emulator.log` | Emulator stdout/stderr — first place to look on failure |
| `.state/trimmed.<avd>` | Marks that guest debloat already ran |
| `~/.config/.android/avd/` or `$ANDROID_AVD_HOME` | AVD definitions |

`./droidlite status --json` returns:

```json
{"running":true,"serial":"emulator-5554","flavor":"gapps","booted":true,
 "guest":{"free_mb":1722,"total_mb":2473},"host_rss_mb":2100}
```

## Tasks

### Test an APK end to end

```bash
./droidlite start --headless
./droidlite inspect app-debug.apk          # package + ABIs + minSdk
./droidlite install app-debug.apk
./droidlite run "$(./droidlite inspect app-debug.apk | sed -n "s/.*name='\([^']*\)'.*/\1/p")"
./droidlite shot /tmp/after-launch.png
./droidlite logcat -d > /tmp/app.log
./droidlite stop
```

### Instrumentation tests

```bash
./droidlite start --headless
adb -s emulator-5554 shell am instrument -w com.example.test/androidx.test.runner.AndroidJUnitRunner
./droidlite stop
```

### Get a screenshot without a window

`./droidlite shot out.png` uses `adb exec-out screencap -p`; no GPU window is
needed.

### Debug a crash

```bash
./droidlite logcat -d | grep -iE 'FATAL|AndroidRuntime|UnsatisfiedLink'
./droidlite shell run-as com.example.app ls
```

## Guardrails

- **Never `reset` without asking.** It wipes the guest's user data.
- **Check ABIs before installing.** `inspect` prints `native-code`. AOSP images
  cannot load `arm64-v8a`-only libraries; `--image gapps` includes NDK
  translation and can (slower). For native speed, rebuild with x86_64.
- **Do not edit the Android SDK** under `$ANDROID_SDK_ROOT` except via
  `setup`/`prune`.
- **Prefer headless + `shot`** on machines with < 8 GB RAM; SwiftShader avoids
  the host GPU.
- **One instance only.** The port is fixed at 5554; stop before changing images.
- The guest is disposable. If it is wedged: `./droidlite stop`, then
  `./droidlite reset`, then start again.

## Failure decoding

| Message | Cause | Action |
|---|---|---|
| `KVM device present` FAIL | no `/dev/kvm` or no permission | stop; host cannot accelerate |
| `boot did not finish in 240s` | slow host / bad image | inspect `.state/emulator.log`, retry with `--profile normal` |
| `emulator exited during boot` | GPU/display or image error | read log tail; try `--headless` |
| `INSTALL_FAILED_NO_MATCHING_ABIS` | APK lacks x86_64 | use gapps or rebuild |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | signature change | `adb -s emulator-5554 uninstall <pkg>` or `reset` |
| `device unauthorized` | stale ADB key | `adb kill-server`, restart emulator |

## Environment

| Variable | Default | Purpose |
|---|---|---|
| `ANDROID_SDK_ROOT` | auto-discovered | SDK location |
| `DROIDLITE_API` | `35` | Android API level |
| `DROIDLITE_IMAGE` | `aosp` | `aosp`, `atd`, `gapps` |
| `DROIDLITE_PORT` | `5554` | ADB console port |

## Design intent

Keep the guest small, keep the CLI boring, keep the device a normal ADB target.
If a change adds a service, sensor, radio or UI chrome to the guest, it is
probably wrong for this project.
