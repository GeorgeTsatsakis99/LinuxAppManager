# App Manager

A small GTK app for Linux that shows you **everything installed on your machine**: apt packages, snaps, flatpaks, Windows programs running under Wine or Bottles, browser web apps, AppImages, command-line tools and random stuff you dropped in `/opt`. It lets you uninstall things without having to remember which package manager they came from, and cleans up what uninstalls leave behind.

I wrote it because on my Zorin box I kept doing the same routine: "is this thing a flatpak or a .deb? what's the package even called? will `apt remove` take half my desktop with it?". The distro's software center only knows about the software it installed itself, and Synaptic shows 3,000 packages with names like `libgnome-desktop-4-2t64`. I wanted one window that answers *what's on this computer* and makes removing things hard to get wrong.

It's a single Python file with no dependencies beyond what a GNOME-based desktop already ships.

![Installed programs](docs/screenshots/programs.png)

---

## What it does

**Layout**

A sidebar on the left (Applications, one entry per source, All packages, Clean up, Startup apps and System monitor, each with a live count, plus a card showing the machine's hostname, OS and when it was last scanned). The page in the middle has a title, stats ("102 apps · 11.2 GB on disk") and search. On the right, a details panel for whatever you selected. Status messages pop up as a small toast at the bottom instead of a dialog.

**Programs**

- Lists every app the desktop menu can show, with its real name, icon and description, not the package name.
- Covers all the places software comes from:
  - **APT / dpkg**: every installed package (not just the ones you installed by hand), each marked *installed by you* or *came with the system*
  - **Snap**
  - **Flatpak** (system and user installations)
  - **Flatpak runtimes** (in *All packages*, read-only: they're what `flatpak uninstall --unused` is for)
  - **Other**: things no system package manager owns:
    - **Windows programs** in every Wine prefix and Bottles bottle, read from the prefix's own *Add/Remove programs* registry, so agents and tools with no menu entry show up too, merged with their Start-menu entries (WPS Office's six launchers become one app)
    - browser web apps (Brave/Chrome "install as app"), **AppImages** (even ones with no launcher, e.g. sitting in `~/Documents`), your own scripts with a `.desktop` file
    - vendor installs in `/opt` and `/usr/local` (netdata, ollama, VMware…), with the vendor's own uninstaller if it ships one
    - command-line tools: `~/.local/bin`, `pipx`, `npm -g`, `cargo install`, rustup toolchains
    - **leftovers**: launchers whose program is gone, Windows programs whose folder is gone (or holds only `unins000.exe`), empty folders in `/opt`
- **Applications** shows the ~100 things that are actually apps; **Unmanaged** everything no package manager tracks (each with how to remove it); **All packages** all ~3,000 entries, libraries and runtimes included
- Search (name, description, package id, and the names of extra apps a package bundles) and sort by name, by size on disk, or by **recently installed**
- Details panel: source / "Installed by you" / "Preinstalled" / "Protected" chips, **Open**, **Show files**, **Uninstall**, and a property list (package, version, install date, size, location…)
- Uninstall runs through `pkexec`, so you get the normal system password prompt, and the package manager's output streams into the window while it works
- **Things no package manager owns get their own way out**, where one is safe:
  - **Move to Trash** for AppImages (with their launchers), leftover launchers and your own scripts in `~/.local/bin`; **Remove from menu** for your own programs' launchers. It's reversible, and the toast offers **Undo**.
  - **Run its uninstaller** for a Windows program that ships one, otherwise **Open Wine uninstaller** (in the right prefix), or **Open Bottles** for programs inside a bottle
  - **Uninstall** for `pipx`, `npm -g` (your own Node), `cargo install` and rustup toolchains, by running that tool's own uninstall command as you, after showing you the exact command
  - Web apps, `/opt` and `/usr/local` installs only say how to remove them (that needs the browser, or root plus the vendor's steps)
- **Opens instantly**: the window shows the last scan straight away and refreshes it in the background ("updating…"). Buttons that change anything stay disabled until the fresh scan is in.

**Clean up**

- One page with everything nothing needs any more, each with its size:
  - **Unused packages**: what `apt autoremove` would remove (here: 37 Qt/Wireshark libraries, 193 MB, left by an earlier uninstall)
  - **Leftover settings** of packages that are already uninstalled (`rc` in dpkg), including the module folders old kernels leave in `/lib/modules`
  - **Downloaded package files** APT keeps in `/var/cache/apt/archives`
  - **Unused Flatpak runtimes**, decided by Flatpak's own rules
  - **Leftovers from removed apps** found by the scan (dead launchers, Windows programs whose folder is gone, empty `/opt` folders). Click one to jump to it and its fix.
- Every button first shows the exact list. Right before running, the app checks that the list still matches (and blocks anything that would touch an essential package). Then it runs one `pkexec` command.
- After you uninstall a `.deb`, the toast offers a jump to Clean up, because that's when packages become unused.

**Startup apps**

- What starts at login, with a switch per entry, and **what really runs**: entries that never start on your desktop (`OnlyShowIn`/`NotShowIn`, a missing `TryExec` program) are marked "Not used on GNOME" and have no switch, and condition-gated ones (Orca) say so
- Background services (the ~30 `NoDisplay` entries GNOME's own Startup Applications hides) are behind **Show background services**. Anything you added or changed is always listed, so you can always undo it.
- Services your desktop starts in an early session phase (gnome-settings-daemon plugins, the keyring) are marked **Part of the desktop**, and turning one off asks first
- Turning off a *system* entry doesn't touch `/etc`. It writes a per-user override in `~/.config/autostart`, which is how GNOME expects it to be done, and you can switch it back on at any time. Such an override is shown as **Changed by you**, not as a separate entry.
- Entries you added yourself can be deleted. They go to the Trash, with **Undo**.

**System monitor**

- Headline tiles: CPU usage, CPU temperature, memory in use and GPU temperature
- Two charts of the last 2 minutes, one for CPU usage and one for CPU temperature. Hover them to read any point.
- Per-thread usage and clock speed, memory / swap / free disk space, and **every temperature sensor** the kernel exposes (CPU package and cores, chipset, NVMe, motherboard…)
- How many times the processor has slowed itself down to stay cool since boot (on Intel). On a fanless laptop that number says more than the temperature does.
- Graphics: clock speed, usage and video memory where the driver reports them, plus the temperature
- The busiest programs, with each program's processes added together (Brave's 18 processes show up as one "brave" line)
- Warm, hot and nearly-full readings turn amber or red, and always come with a warning icon and a word, so you don't have to rely on colour
- It only samples while the page is open, and the sidebar shows the CPU temperature while it does

![System monitor](docs/screenshots/monitor.png)

| | |
|---|---|
| ![Clean up](docs/screenshots/cleanup.png) | ![Startup apps](docs/screenshots/startup.png) |
| ![Confirm dialog](docs/screenshots/confirm-uninstall.png) | ![Protected package](docs/screenshots/protected.png) |
| ![Windows program](docs/screenshots/windows-app.png) | ![All packages](docs/screenshots/all-packages.png) |

---

## Safety: how it avoids breaking your system

Uninstalling is the one destructive thing this app does, so most of the code around it is there to stop you from shooting yourself in the foot. The rules:

1. **apt removals are simulated first.** Before you're even asked to confirm, it runs `apt-get -s remove <pkg>` (a dry run, no root needed) and shows you **every** package that would go, not just the one you clicked. apt loves to cascade: removing one innocent-looking package can drag the whole desktop metapackage with it.

2. **Protected packages are refused outright.** If the dry run touches anything on the essential list, the app won't do it, full stop. The list (see `_PROTECTED_RE` in the source) covers the base system and the desktop: `zorin-*`, `ubuntu-desktop*`, `gnome-shell`, `gnome-session*`, `gdm3`, `mutter`, `nautilus`, Xorg/Wayland, `systemd*`, `libc6`, `bash`, `dash`, `coreutils`, `dpkg`, `apt`, PAM, polkit/`pkexec`, `sudo`, D-Bus, NetworkManager, plymouth, GRUB, kernel images/headers/firmware, GTK/GLib and `python3` itself. For snaps: `snapd`, the `core*` bases, `bare`, and the GNOME/GTK theme snaps. Protected rows show a lock icon and the Uninstall button is disabled.

3. **Nothing runs through a shell.** Every command is built as an argv list and passed to `subprocess` directly. No `shell=True`, no string formatting into commands, so a package name with weird characters can't turn into command injection.

4. **No stored passwords, no running as root.** The GUI runs as you. Only the actual removal command is elevated, via `pkexec`, one command at a time.

5. **`remove` is the default, `purge` is opt-in.** The checkbox for deleting system-wide config files starts unticked. Your home folder is never touched by apt anyway.

6. **Cancel is the default button** in the confirm dialog, so hammering Enter won't uninstall anything.

7. **Things no package manager owns are only removed reversibly, or by their own uninstaller.** The app never deletes such files outright. *Move to Trash* only accepts files inside your home folder (no folders, nothing system-wide) and can be undone. Windows programs are handed to their own uninstaller or Wine's. `pipx`/`npm`/`cargo`/`rustup` run their own uninstall command as you, never as root, after showing the exact command. Everything else (web apps, `/opt`, `/usr/local`) just tells you how.

8. **Clean-up actions are re-checked right before they run.** The list you confirmed may be minutes old, so apt is asked again (`apt-get -s remove …` / `-s purge …`). If the set changed, or would now touch an essential package, nothing happens and you're told why. Leftover settings of the *running* kernel are never offered.

9. **Startup changes are user-level and reversible.** System entries are only ever switched off for your account, never deleted. Session-critical ones ask before turning off. Your own entries go to the Trash.

What it does **not** protect you from: removing an app you actually wanted. It will happily uninstall Firefox if you click through the dialog.

---

## Requirements

- Linux with GTK 3 (developed on **Zorin OS 18**, Ubuntu 24.04 base, GNOME on Wayland)
- Python 3.8 or newer should be fine; I develop and test on 3.12
- PyGObject (`python3-gi`) with the GTK 3 typelib (`gir1.2-gtk-3.0`)
- `pkexec` (polkit) for uninstalling
- Optional, depending on what you use: `dpkg`/`apt`, `snap`, `flatpak`. Missing ones are just skipped.

On a stock Ubuntu/Zorin/Mint desktop all of this is already installed. If not:

```bash
sudo apt install python3-gi gir1.2-gtk-3.0 policykit-1
```

No pip packages, no virtualenv.

## Install & run

```bash
git clone https://github.com/<you>/app-manager.git
cd app-manager
python3 app-manager.py
```

To get it in your app menu, create `~/.local/share/applications/app-manager.desktop`:

```ini
[Desktop Entry]
Type=Application
Name=App Manager
Comment=List, uninstall, and manage startup of installed programs (safe)
Exec=python3 /full/path/to/app-manager/app-manager.py
Icon=system-software-install
Terminal=false
Categories=System;Settings;PackageManager;
Keywords=uninstall;remove;programs;apps;startup;autostart;packages;
```

(Use an absolute path in `Exec=`; `~` isn't expanded there.)

## Keyboard shortcuts

| Key | Action |
|---|---|
| just start typing | search (on any programs page) |
| `Ctrl+F` | focus search |
| `Delete` | uninstall (or, for things no package manager owns, remove) the selected item, always after a confirm |
| `F5` / `Ctrl+R` | rescan the computer (on Clean up: check again; on System monitor: take a reading now) |
| `Ctrl+1` … `Ctrl+9` | jump to a sidebar section |

## Command line

The backend works without a display, which is handy over SSH or in scripts:

```bash
python3 app-manager.py --list                 # everything installed, one line each
python3 app-manager.py --list --apps          # only apps (incl. Windows programs, web apps, AppImages)
python3 app-manager.py --list --apps --json   # same, as JSON
python3 app-manager.py --check gparted        # what would `apt remove gparted` take with it?
python3 app-manager.py --monitor              # one system-monitor snapshot (takes ~1 s)
python3 app-manager.py --monitor --json       # same, as JSON
python3 app-manager.py --startup              # what starts at login, and what really runs
python3 app-manager.py --cleanup              # what could be cleaned up, item by item (changes nothing)
```

`--monitor` output on my laptop:

```
CPU      Intel Core i7-7Y75 @ 1.30GHz — 4 threads, 25% busy, 2.1 GHz avg (max 3.6 GHz)
         72 °C (CPU package) · slowed down to cool off 817 times since boot
Load     4.42 3.76 3.42 · up 1 h 41 min
Memory   5.0 GB of 7.6 GB used · swap 509 MB of 5.8 GB
Disk     System (/): 88.5 GB free of 221.3 GB
GPU      Intel HD Graphics 615 (i915, integrated) — 300 of 1050 MHz · 72 °C (CPU package sensor)
Temps    CPU package 72 °C, Core 0 72 °C, Core 1 71 °C, Chipset 64 °C, Motherboard 28 °C
Top      brave 11.9%, gnome-shell 3.4%, gnome-system-monitor 1.7%, apps.plugin 1.5%, netdata 1.2%
```

`--check` exit codes, so you can use it in scripts:

| code | meaning |
|---|---|
| 0 | removal is fine, nothing essential in the set |
| 1 | package not installed / nothing to remove |
| 2 | usage error |
| 3 | **blocked**: the removal would take an essential package with it |

```
$ python3 app-manager.py --check gnome-shell
...
BLOCKED: this would remove essential package(s): zorin-os-desktop, gdm3, zorin-desktop-session, gnome-shell, ...
$ echo $?
3
```

---

## How it works

### Finding what's installed

Package managers only know about their own packages, and none of them knows which package is "an app". So the app works from two sides and joins them:

**1. Ask each package manager what it has**

| Source | Command | Notes |
|---|---|---|
| apt | `dpkg-query -W` | Only status `ii` (installed). `rc` = removed but config left behind is skipped (Clean up offers those). "Installed by you" vs "came with the system" comes from APT's own state file, `/var/lib/apt/extended_states`. That's the same answer `apt-mark showmanual` gives (296 = 296 here), but in ~0.05 s instead of 1.5–7 s. The install date is when dpkg last wrote the package's file list. |
| snap | `snap list` | Size = sum of all stored revisions in `/var/lib/snapd/snaps`, since removing a snap frees all of them. |
| flatpak | `flatpak list --app --columns=…` | The size column is locale-formatted (`2,1 MB` with a non-breaking space on a Greek locale, which took me a while to notice), so it gets parsed rather than trusted. |
| flatpak runtimes | `flatpak list --runtime` | Listed under *All packages* only, and protected: removing a runtime an app needs breaks the app. |

**2. Read every launcher (`.desktop` file) the desktop would show**

The app walks the XDG application directories in the same precedence order the menu uses: `~/.local/share/applications` first, then everything in `$XDG_DATA_DIRS` (flatpak exports, snap's desktop dir, `/usr/local/share`, `/usr/share`). **Sub-folders count**: per the spec `wine/Programs/AnyDesk/AnyDesk.desktop` is the desktop-id `wine-Programs-AnyDesk-AnyDesk.desktop`, and that's where Wine puts every Windows program's menu entry (an early version only read the top level and missed all of them). A launcher in your home folder with the same desktop-id as a system one **overrides** it, just like in the real menu. Launchers with `NoDisplay`/`Hidden`, a `TryExec` program that isn't installed, or excluded by `OnlyShowIn`/`NotShowIn` for your desktop, are ignored.

**3. Match each launcher to an owner**

- A launcher whose `Exec=` just runs `flatpak run <app-id>` / `snap run <name>` → that flatpak or snap, whoever wrote the file (Zorin ships its own `com.usebottles.bottles.desktop` that starts the Bottles flatpak; it used to be credited to a `.deb` of desktop files)
- Launchers in flatpak's export dir, or a user override with a flatpak's app id as its name → that flatpak
- Launchers in snap's desktop dir → the snap named in the file (`<snap>_<app>.desktop`)
- Everything else → one batched `dpkg-query -S` call over all launcher paths to find the owning `.deb`
- Still no owner? Parse the launcher's `Exec=` line (strip `%U`-style field codes and `env VAR=…` prefixes, resolve through `$PATH`) and ask dpkg who owns *that binary*. This catches packaged apps whose launcher you created by hand.
- A launcher that runs `wine … something.lnk/.exe` (or a Bottles `-b <bottle>` entry) → set aside for step 4
- Still nothing → it's an **Other** app, classified from its `Exec=` line: web app (`--app-id=`), AppImage, something in your home folder, a manual install, or a launcher pointing at a program that no longer exists.

If one package ships several launchers (Maltego plus its Java config tool, or Zorin's Wine bundle with *Browse C: Drive* / *Configure Wine* / *Uninstall Wine Software*), the best match becomes the row's name, and the rest are listed under **Includes** and are searchable.

**4. Windows programs (Wine, Bottles)**

Wine only creates a menu entry for Start-menu shortcuts, so menus alone miss agents, services and anything installed without a shortcut. Each prefix (`~/.wine`, `$WINEPREFIX`, `~/.local/share/wineprefixes/*`, every Bottles bottle, PlayOnLinux) has the real list in its registry files: the `…\CurrentVersion\Uninstall\*` keys in `system.reg`/`user.reg`, which is exactly what `wine uninstaller` shows. The app reads those (skipping `SystemComponent` entries, updates and Wine's own Mono/Gecko), then:

- groups a program's menu entries (same Start-menu folder, or working folders inside each other) and joins them to the registry entry by name, or by folder when the names differ
- finds the program's folder from `InstallLocation`, else the uninstaller's or icon's folder, else a `Program Files` folder named like the program or `Publisher\Program` (MSI installs often record no path), never one another entry already owns
- sizes that folder, and flags **leftovers**: the folder it names is gone, or all that's left is its uninstaller
- marks Visual C++ / .NET / Access-engine runtimes as runtimes, not apps

**5. Software with no launcher at all**

AppImages anywhere in your home folder (3 levels deep, hidden and build folders skipped), `~/.local/bin` and `/opt`; folders in `/opt` and files in `/usr/local/{bin,sbin}` that dpkg doesn't own (root-only folders there are service state, like containerd's, and are skipped; a folder with nothing runnable left is a leftover); executables in `~/.local/bin`, `~/bin` and `~/go/bin` (several names for one file make one row); `pipx` venvs, global `npm` packages (system and every nvm Node), `cargo install` crates and rustup toolchains.

Whole scan on my machine (~3,000 entries, ~100 apps, sizing about 15 GB of unpackaged software): **6–9 s** on this fanless laptop while a browser keeps both cores busy, against 12–16 s for the old version that found far less. Most of the gain is reading APT's state file instead of running `apt-mark`, and asking dpkg about launcher files and the programs they run in **one** `dpkg-query -S` call (each call spends ~0.7 s just loading dpkg's database). The package-manager listings run in parallel. The folder walks don't, because Python threads gain nothing on them (the GIL): tried, it was slower.

The result is saved to `~/.cache/app-manager/scan.json` (only readable by you), and the next launch shows it immediately while the fresh scan runs.

### Clean up

| Category | How it's found | What the button runs |
|---|---|---|
| Unused packages | `apt-get -s autoremove` (a dry run) | `pkexec apt-get remove -y <exactly those packages>` |
| Leftover settings | dpkg status `rc`, minus anything of the running kernel. Sizes come from `/lib/modules/<version>` | `pkexec apt-get purge -y <those packages>` |
| Downloaded package files | `*.deb` in `/var/cache/apt/archives` | `pkexec apt-get clean` |
| Unused Flatpak runtimes | `flatpak uninstall --unused`, answered "n" at its own prompt. Flatpak has no dry run, and this way its own rules (pins, extensions) decide | `flatpak uninstall --unused -y` (with `pkexec` for the system installation) |
| Leftovers from removed apps | the scan's leftover flags | nothing directly: each links to its row |

It's computed when you first open the page, because APT's dry run can take 20 s on a busy laptop and isn't worth paying at every start.

### Startup apps

Autostart entries come from `~/.config/autostart` and every `$XDG_CONFIG_DIRS/autostart` (Zorin adds `/etc/xdg/xdg-zorin`). A user file with the same name as a system entry is that entry's *setting*, so it's merged into one row. Deleting it would silently turn the system entry back on, which is what an earlier version's trash button did while its dialog said the opposite. For each entry the app works out whether the session would really start it: `OnlyShowIn`/`NotShowIn` against `$XDG_CURRENT_DESKTOP`, `TryExec`, and `AutostartCondition` (`GSettings schema key`, `if-exists`, `unless-exists`). `X-GNOME-Autostart-Phase` marks the ones the desktop itself needs.

### Icons

Icons come from the launcher's `Icon=` key, either an absolute path or a theme name. One gotcha: if you give GTK a list of icon names (`["org.app.Id", "application-x-executable"]`), it checks *all of them* in the current theme before falling back to `hicolor`. Flatpak icons only live in `hicolor`, so the theme's generic icon won every time. The app resolves the name explicitly with `IconTheme.has_icon()` first.

### Why the list is paged

A GTK 3 `ListBox` is great for pretty rows and terrible at 3,000 of them: switching from "apps only" to "all packages" froze the window for ~10 seconds. Filtering and sorting now happen in plain Python over the item dicts. Only the first 150 matches get real widgets, and more are added when you scroll to the bottom (`edge-reached`). Row widgets are cached, so re-filtering doesn't rebuild them. Toggling the full list now takes ~0.25 s.

### Threads

All the slow stuff (the scan, the apt dry run, the actual uninstall, each system-monitor reading) runs in a worker thread. Results come back to the GTK main loop through `GLib.idle_add`, so the UI never blocks. The uninstall output is streamed line by line into the info bar at the top, so you can see apt doing its thing instead of staring at a spinner.

### System monitor

Everything is read straight from `/proc` and `/sys`, as your normal user. It needs no root, no `lm-sensors` and nothing extra installed.

| Reading | Where it comes from |
|---|---|
| CPU usage | `/proc/stat`. Usage is a rate, so it's the difference between two readings 2 s apart. The first numbers show up about 0.6 s after you open the page. |
| Clock speed | `/sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq` |
| Temperatures | every `temp*_input` under `/sys/class/hwmon` (`coretemp`/`k10temp` for the CPU, `amdgpu`/`nouveau` for graphics cards, `nvme`, `pch_*`, `acpitz`…), with each sensor's own critical limit when it reports one. The thermal zones are only a fallback, because some of them report nonsense (one here claims a constant 125 °C). |
| Slowdowns | `cpu0/thermal_throttle/package_throttle_count` (Intel) |
| Graphics | Intel: `gt_act_freq_mhz` and `gt_RP0_freq_mhz` on the DRM card. AMD: `gpu_busy_percent`, `pp_dpm_sclk`, `mem_info_vram_*`. NVIDIA's own driver has no hwmon sensors, so it asks `nvidia-smi` when that's installed. |
| Busiest programs | `/proc/<pid>/stat`, grouped by program name. `python3 script.py` shows as `script.py`, and kernel threads like `kworker/3:2-i915` group as `kworker`. |
| Program memory | PSS from `/proc/<pid>/smaps_rollup`, which splits shared pages between the processes using them, so a browser's 18 processes aren't counted 18 times (Brave: 1.6 GB, against 2.9 GB if you add up resident memory) |

Two things worth knowing about graphics on laptops:

- **Integrated Intel graphics have no temperature sensor of their own.** They sit on the same chip as the CPU, so the reading shown is the CPU package sensor, and the page says so.
- **A dedicated card that has powered itself down is left alone.** If `power/runtime_status` says `suspended`, the page shows "Asleep" and doesn't read it. Reading its sensors (or running `nvidia-smi`) would wake it up and drain the battery.

Cost: a reading takes ~20 ms on this 2-core laptop. Reading PSS is the expensive part (~170 ms when Brave is open), so each process's memory is re-read at most every 10 s. Sampling only runs while the page is visible and stops as soon as you switch away.

---

## Project layout

```
app-manager.py          everything: backend, CLI and GUI (one file on purpose)
docs/screenshots/       images used in this README
```

Inside `app-manager.py`, top to bottom:

| Section | What's there |
|---|---|
| essential-package guard | `_PROTECTED_RE`, `_PROTECTED_SNAP`, `is_protected()` |
| enumeration | `list_apt()`, `list_snap()`, `list_flatpak()`, `list_all()` |
| launchers | `application_dirs()`, `scan_launchers()`, `attach_launchers()`, `.desktop` parsing, Exec-line parsing |
| removal planning | `simulate_apt_remove()`, `remove_command()` |
| Windows programs | `wine_prefixes()`, `read_uninstall_entries()`, `windows_programs()` |
| unmanaged software | `list_unpackaged()`, `find_appimages()`, `vendor_paths()`, `local_action()`, `trash_paths()`, `restore_from_trash()` |
| clean-up | `cleanup_report()`, `verify_cleanup()` |
| scan cache | `save_scan()`, `load_scan()` |
| autostart | `list_autostart()`, `_autostart_skip_reason()`, `set_autostart_enabled()`, `remove_autostart()` |
| system monitor | `SystemSampler` (one reading = `sample()`), `cpu_model()`, `list_gpus()`, `_hwmon_temps()`, `_friendly_temps()`, `_gpu_reading()` |
| CLI | `--list`, `--apps`, `--json`, `--check`, `--monitor`, `--startup`, `--cleanup` |
| GUI | `_VIEWS` (sidebar sections), `_CSS` (the stylesheet), `_VIZ`/`_METER_CSS` (chart and meter colours), `run_gui()`: window, sidebar, rows, details panel, dialogs, `MeterRow`/`StatTile`/`HistoryChart` |

The backend functions don't import GTK, so you can import the file and use them from another script.

---

## Known limitations

- **Only this machine.** It reads the local system. Scanning a remote box over SSH isn't there (yet).
- **Other users' per-user installs aren't visible.** Flatpaks installed with `--user` by *another* account, or launchers in another user's home, need that user's session (or root) to read. System-wide stuff is covered.
- **Some "Other" things still can't be removed from the app**, on purpose: web apps (use the browser's apps page), and vendor installs in `/opt` or `/usr/local` (they need root plus the vendor's own steps). Symlinked CLI tools like `claude` aren't trashed either, since that would only remove the link.
- **Untested here:** Clean up's parsing of Flatpak's list of unused runtimes, because this machine has none. If the format differs, the card just shows nothing to clean. Also untested is actually running the `pipx`/`npm`/`cargo`/`rustup` uninstalls and the clean-up commands, since I didn't want to remove real software to prove it. Their dialogs and checks were exercised, and the Trash/Undo path was tested end to end on throwaway files.
- **Old snap revisions aren't cleaned up.** snapd keeps two per snap for rollback, and removing them takes one `pkexec` prompt each.
- **Windows-program details are best-effort.** The registry list itself is exact, but matching it to menu entries and finding the program's folder are heuristics (name and folder matching). A program that registered no uninstall entry and has no menu entry (a portable `.exe` you copied in) can't be found. Steam/Proton games are left to Steam.
- **Not covered:** `pip install --user` packages (mostly libraries), Docker/Podman images and containers, Nix and Homebrew, and the Node.js versions nvm itself installed (their global `npm` packages are listed).
- **apt/dpkg-based distros only** for native packages. Fedora (`dnf`/`rpm`), Arch (`pacman`) and openSUSE (`zypper`) aren't supported. Snap and flatpak work anywhere.
- **"Installed by you" is apt's opinion.** It's what `apt-mark showmanual` says, which also counts things the installer marked manual during OS setup.
- **The install date of a `.deb` is "installed or last updated".** dpkg keeps no separate install time.
- **Snap/flatpak dependency cascades aren't previewed** the way apt's are. Removing a flatpak app leaves its runtime in place (`flatpak uninstall --unused` cleans that up), and snap base snaps are protected anyway.
- The protected list is tuned for Ubuntu-family GNOME desktops (Zorin, Ubuntu, Pop!_OS-ish). On KDE or XFCE you'd want to add `plasma-*` / `xfce4-*` etc. to `_PROTECTED_RE`.
- Only tested on Zorin OS 18 (GNOME, Wayland). It should work on Ubuntu 22.04+ and Mint, but I haven't tried.
- **System monitor: the AMD and NVIDIA graphics paths are untested.** This laptop only has Intel graphics. Those code paths follow the kernel's documented sysfs files and `nvidia-smi`'s CSV output, but no real card has been tried yet.
- **No usage percentage for Intel graphics.** The kernel only reports that through `perf`, which needs root. The page shows the graphics clock speed instead.

## Contributing

Issues and PRs welcome. A few ground rules, because this thing deletes software:

- Never build a command with string formatting or `shell=True`. Argv lists only.
- Anything that removes or changes system state must go through a preview + confirm step first.
- If you add a package source (dnf, pacman…), add its essential packages to the guard **in the same PR**.
- Test the `--check` path against something that must be refused (`gnome-shell`, `systemd`, `libc6`) before sending.

Quick sanity check before a PR:

```bash
python3 -m py_compile app-manager.py
python3 app-manager.py --list --apps | tail -1
python3 app-manager.py --check gnome-shell; echo "exit=$?"   # must print BLOCKED, exit=3
```
