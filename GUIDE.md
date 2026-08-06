# 📘 GUIDE — Phone Temperature Checker

A complete walkthrough: **installation, usage, internals, and troubleshooting**
for the `temp_checker` project. This guide assumes you have never used
Termux before.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Prerequisites](#2-prerequisites)
3. [Downloading Official Termux from F-Droid](#3-downloading-official-termux-from-f-droid)
4. [Installing Termux:API](#4-installing-termuxapi)
5. [Granting Required Permissions](#5-granting-required-permissions)
6. [Installing Dependencies](#6-installing-dependencies)
7. [Placing the Script](#7-placing-the-script)
8. [Running the Project](#8-running-the-project)
9. [Understanding the Output](#9-understanding-the-output)
10. [Common Commands](#10-common-commands)
11. [Examples](#11-examples)
12. [Troubleshooting](#12-troubleshooting)
13. [Common Errors](#13-common-errors)
14. [Frequently Asked Questions](#14-frequently-asked-questions)
15. [Tips](#15-tips)
16. [Warnings](#16-warnings)
17. [How It Works (Internals)](#17-how-it-works-internals)
18. [Project Structure](#18-project-structure)
19. [Build Cache Layout](#19-build-cache-layout)
20. [Future Improvements](#20-future-improvements)
21. [License](#21-license)

---

## 1. Introduction

`temp_checker` is a single-file command-line program that:

- Reads your Android phone's **battery temperature** every 3 seconds
- Shows **battery level**, **charging status** and **current draw**
- Color-codes the temperature with four safety thresholds
- Tracks **min / max / average** over the session

It is written in **C**, wrapped in a tiny **POSIX shell script** that
auto-compiles the program on first run, then runs the cached binary.

| Property | Value |
|---|---|
| Language | C99 + POSIX `sh` |
| Dependencies | `clang`, `termux-api` package, Termux:API Android app |
| Storage | One script file (~10 KB after build) |
| Network | None |
| Root | Not required |

---

## 2. Prerequisites

You need:

- An **Android device** (ARM64 recommended — that is what the cached binary targets)
- The **F-Droid** app installed on your phone (free, open-source app store)
- About **5 minutes**

> [!WARNING]
> Do **NOT** use the Google Play Store version of Termux. It is
> deprecated and incompatible with current `termux-api`. Only the
> F-Droid build works with this project.

---

## 3. Downloading Official Termux from F-Droid

1. Open your phone's browser and go to **https://f-droid.org**
2. Download and install the **F-Droid** app store
3. Open F-Droid, let it initialise its repository index
4. Search for **"Termux"**
5. Install **Termux** (package id: `com.termux`)
6. Open Termux once so the home directory is created

If you already have an older Termux installed from the Play Store,
**uninstall it first** — the two builds will conflict.

---

## 4. Installing Termux:API

Termux:API is a **separate Android app** that exposes phone hardware
(sensors, battery, location, camera, …) as command-line tools inside
Termux. `temp_checker` uses it to read battery information.

1. In F-Droid, search for **"Termux:API"**
2. Install the **Termux:API** app (package id: `com.termux.api`)
3. Open the **Termux:API** app **once** — this registers its IPC
   service with Android so Termux can talk to it. You don't need to
   interact with the app's UI.

> [!IMPORTANT]
> If you skip step 3, `termux-battery-status` will return nothing and
> `temp_checker` will print `Could not read battery info.`

---

## 5. Granting Required Permissions

`termux-battery-status` itself requires **no runtime permission** on
modern Android. However, the Termux:API companion app must be allowed
to keep running in the background.

### Disable battery optimisation (strongly recommended)

Aggressive battery savers (Xiaomi MIUI, Huawei EMUI, OnePlus OxygenOS,
Samsung One UI) may kill Termux:API after a few minutes in the
background, causing `temp_checker` to print `read failed (API app
stopped?)` periodically.

To prevent this:

1. **Settings → Apps → Termux:API**
2. **Battery → Unrestricted** (or "Don't optimise")

On Samsung devices, also:
- **Settings → Battery → Background usage limits** → make sure
  Termux:API is not in the "Deep sleeping apps" list.

---

## 6. Installing Dependencies

Inside Termux, run:

```bash
pkg update && pkg upgrade
pkg install clang termux-api
```

This installs:

| Package | Purpose |
|---|---|
| `clang` | C compiler used by the wrapper |
| `termux-api` | Client-side binaries: `termux-battery-status` etc. |

Verify:

```bash
command -v clang termux-battery-status
```

Both should print absolute paths.

Test the battery API directly:

```bash
termux-battery-status
```

Expected output (your values will differ):

```json
{
  "health": "Good",
  "percentage": 41,
  "plugged": "UNPLUGGED",
  "status": "DISCHARGING",
  "temperature": 38.1,
  "current": -522000
}
```

If this command prints JSON, the rest of the project will work.

---

## 7. Placing the Script

Copy the `temp_checker` file anywhere. Common locations:

| Location | Works? | Notes |
|---|---|---|
| `$HOME` (`/data/data/com.termux/files/home`) | ✅ | Best — no FUSE limitations |
| `/sdcard` | ⚠️ | Use `sh` prefix (see below) |
| Android Workspace | ⚠️ | Use `sh` prefix |

No `chmod +x` is strictly required when using `sh`.

---

## 8. Running the Project

### From `$HOME`
```bash
sh ./temp_checker
# or
./temp_checker
```

### From Android Workspace or `/sdcard`
```bash
sh /sdcard/temp_checker
sh /path/to/temp_checker
```

> [!NOTE]
> The `sh` prefix is **mandatory** when the file lives on a FUSE mount
> (Android Workspace, `/sdcard`) because the kernel blocks direct
> `execve()` from those paths.

On the **first run**, the wrapper:

1. Creates `$HOME/.cache/temp_checker/`
2. Extracts the embedded C source to `temp_checker.c`
3. Compiles it with `clang -O2 -Wall -Wextra`
4. `exec`s the binary in place of the shell

Subsequent runs detect that the binary is newer than the script and
skip the build step.

Press **Ctrl+C** to stop.

---

## 9. Understanding the Output

```
============================================
   Phone Temperature Checker  (Ctrl+C = quit)
============================================
  Refreshing every 3 seconds. Press Ctrl+C to stop.

[18:45:30]  38.1°C  Warm  |  DISCHARGING  |  41%  |  522 mA
[18:45:33]  38.1°C  Warm  |  DISCHARGING  |  41%  |  519 mA  |  min 38.1  max 38.1  avg 38.1  (n=2)
```

### Field-by-field

| Field | Meaning |
|---|---|
| `[18:45:30]` | Local time of the reading (HH:MM:SS) |
| `38.1°C` | Battery temperature in degrees Celsius |
| `Warm` | Color-coded label (see below) |
| `DISCHARGING` | `termux-battery-status` status field |
| `41%` | Battery level (0–100) |
| `522 mA` | Absolute current draw in milliamps |
| `min 38.1 max 38.1 avg 38.1` | Session statistics |
| `(n=2)` | Number of samples so far |

### Color levels

| Color | Range | Label | What to do |
|---|---|---|---|
| 🟢 Green | < 35 °C | `Normal` | Nothing — phone is cool |
| 🟡 Yellow | 35 – 40 °C | `Warm` | Light use is fine |
| 🟠 Orange | 40 – 45 °C | `Hot - close heavy apps!` | Drop brightness, switch to Wi-Fi |
| 🔴 Red | > 45 °C | `CRITICAL - let it cool down!` | Stop heavy tasks, let it cool |

### Final summary on exit

When you press Ctrl+C the program prints a session summary:

```
  Session: 47 samples, min 35.2°C, max 44.8°C, avg 38.7°C
```

---

## 10. Common Commands

| Task | Command |
|---|---|
| First run (builds the binary) | `sh /path/to/temp_checker` |
| Force a rebuild | `rm -rf ~/.cache/temp_checker && sh /path/to/temp_checker` |
| Test the underlying API | `termux-battery-status` |
| Uninstall completely | `rm -rf ~/.cache/temp_checker && rm /path/to/temp_checker` |
| Disable colour | `NO_COLOR=1 sh /path/to/temp_checker` |
| Check compiler | `clang --version` |
| Clear Termux package cache | `pkg clean` |

---

## 11. Examples

### Example A — Long gaming session

Run the program in one terminal while playing a heavy game in another:

```bash
sh ~/temp_checker
```

Watch the temperature climb. When it hits the **orange** threshold
(40 °C), the program will warn you to close heavy apps.

### Example B — Logging while charging

```bash
sh ~/temp_checker
```

Plug in the charger. The status field changes from `DISCHARGING` to
`CHARGING`. The current value typically rises significantly.

### Example C — Quick sanity check

```bash
sh ~/temp_checker
```

Press Ctrl+C after one reading — you'll still get the colour-coded
label and the current value.

---

## 12. Troubleshooting

### "termux: not found"

You are not running inside Termux. Open the Termux app, then run the
script from there.

### "permission denied"

You tried to run the script directly (`./temp_checker`) from a path
that lives on a FUSE mount (Android Workspace, `/sdcard`). Always use
the `sh` prefix from those locations.

### "Could not read battery info."

The C source prints this exact message when `popen()` returns nothing
or the `temperature` field is missing. Checklist:

1. Is the **Termux:API Android app** installed? (Not just the
   `termux-api` package — the app is separate.)
2. Did you **open the Termux:API app once** after installing?
3. Is the `termux-api` package installed?
   ```bash
   command -v termux-battery-status
   ```
4. Is the Termux:API app allowed to run in the background?
   See [Granting Required Permissions](#5-granting-required-permissions).
5. Test the command directly:
   ```bash
   termux-battery-status
   ```
   If it returns nothing, the IPC link is broken.

### "API app stopped" mid-session

After running for a while, lines may turn into:
```
[--:--:--]  read failed (API app stopped?)
```
This means Android killed Termux:API in the background. The program
keeps running and will resume readings once the API app is woken
again. To prevent it permanently, disable battery optimisation for
**both** Termux and Termux:API.

### Temperature seems stuck

Android's `BatteryManager` reports temperature at low resolution
(typically 0.1 °C steps, often unchanged for several seconds). This
is an OS-level limitation, not a bug.

### Stale values after a Termux update

If you recently ran `pkg upgrade`, the IPC socket path may have
changed. Fix:
```bash
pkg upgrade termux-api
# Then open the Termux:API app once.
```

---

## 13. Common Errors

| Error message | Cause | Fix |
|---|---|---|
| `termux: not found` | Outside Termux | Open Termux first |
| `permission denied` | FUSE mount blocks direct execution | Use `sh /path/to/file` |
| `clang: not found` | Compiler not installed | `pkg install clang` |
| `Could not read battery info.` | Termux:API not reachable | See [Troubleshooting](#12-troubleshooting) |
| `read failed (API app stopped?)` | Termux:API killed in background | Disable battery optimisation |
| Empty / no output | Stdin not a TTY or API app not registered | Open Termux:API once |

---

## 14. Frequently Asked Questions

**Q: Does this work without an internet connection?**
A: Yes. Nothing in `temp_checker` touches the network.

**Q: Does this work on iOS?**
A: No. Termux is Android-only.

**Q: Will this void my warranty?**
A: No. It is a read-only program — it cannot modify your battery,
firmware, or settings.

**Q: Why is the current value always positive?**
A: Android reports current in signed microamps. Negative = leaving
the battery (discharging), positive = entering (charging). The
program shows the magnitude (`fabs()`) because the direction is
already encoded in the `status` field.

**Q: Why 3 seconds?**
A: It matches Android's typical battery-event cadence. You can change
it in the C source (`const int interval_s = 3;`).

**Q: Can I run multiple instances?**
A: Yes. Each will compile to its own cache directory if you change
the script name; otherwise the second run just reuses the cached
binary.

**Q: Why is the binary not in the repo?**
A: See [Project Structure](#18-project-structure). It is built
locally on first run because the Android Workspace blocks direct
execution of pre-built binaries.

**Q: Is the F-Droid build really required?**
A: Yes. The Play Store build of Termux has been frozen since 2020
and is incompatible with current `termux-api` packages.

---

## 15. Tips

- 🪫 **Low battery?** Run `temp_checker` — when the level drops below
  ~20% the orange/red labels usually appear, confirming throttling.
- 🎮 **Gaming test?** Open the program in one terminal, launch a
  heavy game, and watch the temperature curve.
- 🧊 **Cooling test?** Place the phone in a cool spot and watch
  `avg` drop over a few minutes.
- 📈 **Long-term trend?** The `min / max / avg (n=…)` line is a
  running summary — let it run for an hour to see the full range.
- 🔕 **Quiet run?** `nohup sh ./temp_checker > temp.log 2>&1 &`
  detaches the program and writes a plain log you can `cat` later.

---

## 16. Warnings

> [!CAUTION]
> - `temp_checker` is a **monitor**, not a controller. It cannot cool
>   your phone. The red label is a hint, not an emergency shutdown.
> - If your phone physically feels too hot, **stop using it** —
>   don't wait for the program to tell you.
> - Do not rely on this tool as the only safety check during
>   stress tests or benchmarking sessions.
> - Disabling battery optimisation reduces standby time slightly.
>   Re-enable it when you're done.

---

## 17. How It Works (Internals)

### The architecture in one diagram

```
┌─────────────┐   popen()   ┌──────────────────────┐   IPC   ┌────────────┐
│ temp_checker│ ──────────► │ termux-battery-status│ ──────► │  Termux:API │
│   (C code)  │ ◄────────── │       (JSON)         │ ◄────── │   (app)    │
└─────────────┘   parse     └──────────────────────┘         └────────────┘
       │                                                            │
       ▼                                                            ▼
   print line                                              Android BatteryManager
   every 3 s                                               (system service)
```

### The shell wrapper

`temp_checker` is actually **two things in one file**:

1. A POSIX `sh` prologue (the lines before the first `<<'EOF'` or
   `cat <<EOF` heredoc).
2. The C source code, embedded in a heredoc that the shell writes to
   disk on first run.

When you run `sh temp_checker`, the shell:

1. Detects the heredoc marker.
2. Writes everything between `<<'EOF_C'` and `EOF_C` to
   `$HOME/.cache/temp_checker/temp_checker.c`.
3. Runs `clang -O2 -Wall -Wextra temp_checker.c -o temp_checker_bin`.
4. Replaces the shell with the binary using `exec`.

On every subsequent run, the wrapper compares the modification time
of the script (`$0`) with that of the cached binary using `find -newer`.
If the binary is newer, the build step is skipped entirely.

### The C source — file by file

The C source lives entirely in the heredoc. There are **no external
files**. The functions it defines are:

| Function | Purpose |
|---|---|
| `trim()` | Strips leading/trailing whitespace from a string in place |
| `extract()` | Pulls a numeric or short string value out of `"key":value` JSON pairs |
| `celsius_to_label()` | Maps a float °C to a colour code and a safety label |
| `sigint_handler()` | Async-signal-safe Ctrl+C handler that just sets a flag |
| `main()` | Initialises state, enters a `while (!done)` loop |

### JSON parsing without a library

`termux-battery-status` prints compact JSON with one field per line.
The parser does **not** parse JSON generically — it searches for
known substrings:

```c
char *p = strstr(json, "\"temperature\":");
if (p) {
    p += strlen("\"temperature\":");
    double t = strtod(p, NULL);
    ...
}
```

This is safe here because:

- The keys are fixed and known.
- `termux-battery-status` is a local, trusted binary.
- The string lives only for the duration of one `popen()` call.

A real JSON library would add ~50 KB of binary for parsing four
numbers and two short strings. For this project that's overkill.

### The colour palette

```c
GREEN   = "\033[32m"   // < 35 °C  Normal
YELLOW  = "\033[33m"   // < 40 °C  Warm
ORANGE  = "\033[38;5;208m"  // < 45 °C  Hot
RED     = "\033[31m"   // >= 45 °C Critical
DIM     = "\033[2m"
RESET   = "\033[0m"
```

`ORANGE` uses the 256-colour escape because classic ANSI has no orange.
Most modern terminals understand it. If yours doesn't, that one line
will simply lose colour, not break.

### Signal handling

```c
static volatile sig_atomic_t done = 0;

static void sigint_handler(int sig) {
    (void)sig;
    done = 1;
}
```

`done` is `volatile sig_atomic_t` — the only type guaranteed safe for
async signal access in C. The handler does almost nothing; the main
loop checks the flag at the next safe point.

### Build flags

```sh
clang -O2 -Wall -Wextra -o "$BIN" "$SRC"
```

- `-O2` — basic optimisation; keeps the binary small.
- `-Wall -Wextra` — surfaces anything genuinely wrong. Both should
  yield **zero** warnings on the current source.

### Why `fabs()` for current?

Android's `BatteryManager` reports current in **signed microamps**:

```
discharging:  current = -522000   (battery losing 522 mA)
charging:     current = +612000   (battery gaining 612 mA)
```

The `status` field (`DISCHARGING` / `CHARGING`) already conveys
direction. Showing the magnitude in mA is therefore both shorter and
easier to scan.

```c
double mA = fabs(current_uA) / 1000.0;
```

---

## 18. Project Structure

```
phone-temperature-checker/
├── temp_checker              # The whole program: sh wrapper + embedded C
├── README.md                 # Quick start, features, troubleshooting
├── GUIDE.md                  # This file — full walkthrough
├── LICENSE                   # MIT
└── assets/
    ├── banner.svg            # README header
    ├── social-preview.svg    # GitHub social preview (1280×640)
    ├── icons/
    │   ├── battery.svg
    │   ├── temperature.svg
    │   ├── android.svg
    │   ├── terminal.svg
    │   ├── c.svg
    │   ├── health.svg
    │   ├── charging.svg
    │   ├── voltage.svg
    │   ├── current.svg
    │   └── power.svg
    └── illustrations/
        ├── dashboard.svg
        ├── installation.svg
        └── documentation.svg
```

There is **no** `Makefile`, `CMakeLists.txt`, or `configure` script.
The wrapper builds the binary on first run.

---

## 19. Build Cache Layout

```
$HOME/.cache/temp_checker/
├── temp_checker.c      # extracted C source
└── temp_checker_bin    # cached binary
```

To force a clean rebuild:

```bash
rm -rf "$HOME/.cache/temp_checker"
sh /path/to/temp_checker
```

To uninstall:

```bash
rm -rf "$HOME/.cache/temp_checker"
rm /path/to/temp_checker
```

The cached binary is architecture-specific. If you copy the script to
a second device, delete the cache first so it recompiles natively.

---

## 20. Future Improvements

Listed as future work only (none are currently implemented):

- CLI flags: `--interval 5`, `--no-color`, `--csv log.csv`
- Configurable alert threshold with `termux-notification` and `termux-vibrate`
- `NO_COLOR=1` support (already partially wired up)
- Sparkline of the last N readings
- CPU temperature on rooted devices (`/sys/class/thermal/`)
- Auto-rebuild when `clang` major version changes
- Optional GNUplot/ASCII chart export

---

## 21. License

Released under the **[MIT License](LICENSE)**. © 2026 Shyam.

```
MIT License

Copyright (c) 2026 Shyam

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

<div align="center">

Made with ❤️ for the Termux community.

</div>
