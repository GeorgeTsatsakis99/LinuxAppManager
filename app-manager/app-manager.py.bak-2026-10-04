#!/usr/bin/env python3
"""App Manager — a safe GTK GUI to list, uninstall, and manage startup of programs on Linux.

Three areas:
  • Installed Programs — enumerate apps from apt / snap / flatpak, search & filter, and UNINSTALL
    one at a time. Uninstalling is destructive and can cascade, so this is safe-by-default:
      - apt removals are SIMULATED first (`apt-get -s remove`) and the full set of packages that
        would be removed is shown for confirmation;
      - an essential/desktop-critical guard refuses removals that would take out the base system
        (zorin*, gnome-shell, gdm3, xorg, systemd, the kernel, apt/dpkg, libc6, …);
      - privileged actions run through `pkexec` (a graphical auth prompt) — no stored passwords;
      - `remove` (not `purge`) is the default; purge is an explicit opt-in for apt.
  • Startup Apps — the "edit" side: enable/disable/remove autostart entries. This is user-level
    (no root) and is the safe way to cut what runs at login (and the CPU it costs).
  • System monitor — live CPU / memory / disk use, CPU and GPU temperatures, graphics clock and
    the busiest programs. Read-only from /proc and /sys (no root); sampled only while visible.

Backend is usable headlessly for verification / scripting:
    app-manager.py --list [--apps] [--json]
                                       list everything installed on the machine (--apps: only
                                       programs with a launcher, incl. web apps / AppImages)
    app-manager.py --check <pkg>       show what an apt removal of <pkg> would take, and whether
                                       it is blocked by the essential-package guard
    app-manager.py --monitor [--json]  one system-monitor snapshot (CPU, memory, temperatures,
                                       GPU, busiest programs)
Run with no arguments to launch the GUI.
"""
from __future__ import annotations

import collections
import glob
import json
import os
import re
import shlex
import shutil
import subprocess
import sys
import threading
import time

# ── essential / desktop-critical packages the uninstaller must never remove ───────────────────
# Removing any of these (directly or via cascade) would break the desktop or the base system.
_PROTECTED_RE = re.compile(
    r"^(zorin[\w.-]*|ubuntu-desktop[\w.-]*|gnome-shell|gnome-session[\w.-]*|gdm3|mutter[\w.-]*|"
    r"nautilus[\w.-]*|xorg|xserver-xorg[\w.-]*|xwayland|wayland[\w.-]*|systemd[\w.-]*|init|"
    r"libc6[\w.-]*|bash|dash|coreutils|dpkg|apt|apt-utils|libapt[\w.-]*|libpam[\w.-]*|"
    r"policykit-1|polkitd|pkexec|sudo|dbus[\w.-]*|network-manager[\w.-]*|plymouth[\w.-]*|"
    r"grub[\w.-]*|linux-image[\w.-]*|linux-generic[\w.-]*|linux-headers[\w.-]*|"
    r"linux-firmware|libgtk[\w.-]*|libglib[\w.-]*|python3(\.\d+)?|perl-base)$"
)
_PROTECTED_SNAP = {"snapd", "core", "core18", "core20", "core22", "core24", "bare",
                   "gnome-42-2204", "gtk-common-themes"}


def is_protected(name: str, source: str = "apt") -> bool:
    if source == "snap":
        return name in _PROTECTED_SNAP or name.startswith(("gnome-", "gtk-", "kde-"))
    return bool(_PROTECTED_RE.match(name))


def _run(argv: list[str], timeout: int = 120) -> tuple[int, str, str]:
    """Run a command safely (argv list, never a shell string). Returns (rc, stdout, stderr)."""
    try:
        p = subprocess.run(argv, capture_output=True, text=True, timeout=timeout, check=False)
        return p.returncode, p.stdout, p.stderr
    except FileNotFoundError:
        return 127, "", f"{argv[0]}: not found"
    except subprocess.TimeoutExpired:
        return 124, "", f"{argv[0]}: timed out"


def _human_kb(kb: int) -> str:
    mb = kb / 1024
    if mb >= 1024:
        return f"{mb/1024:.1f} GB"
    return f"{mb:.0f} MB" if mb >= 1 else (f"{kb} KB" if kb else "-")


# ── enumeration (the whole machine, not just what you installed) ─────────────────────────────
def list_apt() -> list[dict]:
    """Every installed dpkg package. `manual` marks the ones a person explicitly asked for; the
    rest came with the OS or as dependencies."""
    if not shutil.which("dpkg-query"):
        return []
    manual = set(_run(["apt-mark", "showmanual"])[1].split())
    fmt = ("${db:Status-Abbrev}\t${binary:Package}\t${Version}\t${Installed-Size}\t"
           "${binary:Summary}\n")
    _, out, _ = _run(["dpkg-query", "-W", "-f", fmt])
    rows = []
    for line in out.splitlines():
        parts = line.split("\t")
        if len(parts) < 4 or parts[0][1:2] != "i":  # skip removed-but-config-left ("rc") etc.
            continue
        ident, ver, size = parts[1], parts[2], parts[3]
        pkg = ident.split(":")[0]
        try:
            kb = int(size)
        except ValueError:
            kb = 0
        rows.append({"source": "apt", "id": ident, "name": pkg, "version": ver,
                     "size": _human_kb(kb), "size_kb": kb,
                     "summary": parts[4] if len(parts) > 4 else "",
                     "manual": pkg in manual or ident in manual,
                     "protected": is_protected(pkg)})
    return rows


def list_snap() -> list[dict]:
    if not shutil.which("snap"):
        return []
    _, out, _ = _run(["snap", "list"])
    rows = []
    for line in out.splitlines()[1:]:  # skip header
        f = line.split()
        if len(f) < 2:
            continue
        name, ver = f[0], f[1]
        notes = " ".join(f[5:]) if len(f) > 5 else ""
        rows.append({"source": "snap", "id": name, "name": name, "version": ver,
                     "size": "-", "summary": f"snap ({notes})" if notes and notes != "-" else "",
                     "protected": is_protected(name, "snap")})
    return rows


def list_flatpak() -> list[dict]:
    if not shutil.which("flatpak"):
        return []
    cols = "application,name,version,size,installation"
    _, out, _ = _run(["flatpak", "list", "--app", f"--columns={cols}"])
    rows = []
    for line in out.splitlines():
        f = line.split("\t")
        if len(f) < 2 or not f[0]:
            continue
        app_id = f[0]
        rows.append({"source": "flatpak", "id": app_id, "name": f[1] or app_id,
                     "version": f[2] if len(f) > 2 else "",
                     "size": f[3] if len(f) > 3 else "-",
                     "installation": f[4] if len(f) > 4 else "user",
                     "summary": "", "protected": False})
    return rows


def list_all() -> list[dict]:
    rows = list_apt() + list_snap() + list_flatpak()
    attach_launchers(rows)
    rows.sort(key=lambda r: (r["source"], r["name"].lower()))
    return rows


# ── launchers: every app the desktop menu shows, mapped to whatever owns it (read-only) ─────────
_SNAP_APPDIR = "/var/lib/snapd/desktop/applications"
_FLATPAK_APPDIRS = ("/var/lib/flatpak/exports/share/applications",
                    os.path.expanduser("~/.local/share/flatpak/exports/share/applications"))


def application_dirs() -> list[str]:
    """XDG application dirs in desktop-menu precedence order (user first), de-duplicated."""
    home = os.environ.get("XDG_DATA_HOME") or os.path.expanduser("~/.local/share")
    data = (os.environ.get("XDG_DATA_DIRS") or "/usr/local/share:/usr/share").split(":")
    extra = [os.path.dirname(d) for d in _FLATPAK_APPDIRS] + [os.path.dirname(_SNAP_APPDIR)]
    dirs, seen = [], set()
    for base in [home, *data, *extra, "/usr/local/share", "/usr/share"]:
        d = os.path.realpath(os.path.join(base, "applications")) if base else ""
        if d and d not in seen and os.path.isdir(d):
            seen.add(d)
            dirs.append(d)
    return dirs


def _locale_keys(key: str) -> list[str]:
    lang = (os.environ.get("LANGUAGE") or os.environ.get("LC_ALL")
            or os.environ.get("LC_MESSAGES") or os.environ.get("LANG") or "")
    lang = lang.split(":")[0].split(".")[0].split("@")[0]
    keys = []
    if lang and lang not in ("C", "POSIX"):
        keys.append(f"{key}[{lang}]")
        if "_" in lang:
            keys.append(f"{key}[{lang.split('_')[0]}]")
    keys.append(key)
    return keys


def read_desktop(path: str) -> dict:
    """Parse the [Desktop Entry] group of a .desktop file into a flat dict."""
    entry, in_group = {}, False
    try:
        with open(path, encoding="utf-8", errors="replace") as fh:
            for line in fh:
                line = line.strip()
                if line.startswith("["):
                    in_group = line == "[Desktop Entry]"
                    continue
                if in_group and "=" in line and not line.startswith("#"):
                    k, v = line.split("=", 1)
                    entry[k.strip()] = v.strip()
    except OSError:
        pass
    return entry


def _localized(entry: dict, key: str) -> str:
    for k in _locale_keys(key):
        if entry.get(k):
            return entry[k]
    return ""


def _visible_launcher(entry: dict) -> bool:
    if (entry.get("NoDisplay", "").lower() == "true" or entry.get("Hidden", "").lower() == "true"
            or entry.get("Type", "Application") != "Application"):
        return False
    desktops = {d.lower() for d in os.environ.get("XDG_CURRENT_DESKTOP", "GNOME").split(":")}
    only = {d.lower() for d in entry.get("OnlyShowIn", "").split(";") if d}
    never = {d.lower() for d in entry.get("NotShowIn", "").split(";") if d}
    return (not only or bool(only & desktops)) and not (never & desktops)


def _apply_launcher(row: dict, path: str, entry: dict) -> None:
    row["name"] = _localized(entry, "Name") or row["name"]
    row["summary"] = _localized(entry, "Comment") or row.get("summary", "")
    row["icon"] = entry.get("Icon", "")
    row["desktop"] = path
    row["is_app"] = True


def _parse_size_kb(text: str) -> int:
    """'2,1 MB' / '607.7 MB' / '1.2 GB' (locale commas, NBSPs) -> KiB."""
    m = re.match(r"\s*([\d.,]+)\s*([kKMGT]?)i?B", (text or "").replace(" ", " "))
    if not m:
        return 0
    try:
        num = float(m.group(1).replace(",", "."))
    except ValueError:
        return 0
    return int(num * {"": 1 / 1024, "k": 1, "K": 1, "M": 1024,
                      "G": 1024 ** 2, "T": 1024 ** 3}[m.group(2)])


def _exec_argv(entry: dict) -> list[str]:
    """Exec= split into argv with field codes (%U, %f…) and a leading `env VAR=…` removed."""
    try:
        argv = shlex.split(entry.get("Exec", ""))
    except ValueError:
        argv = entry.get("Exec", "").split()
    argv = [a for a in argv if not re.fullmatch(r"%[a-zA-Z]", a)]
    if argv and os.path.basename(argv[0]) == "env":
        argv = argv[1:]
        while argv and (("=" in argv[0] and not argv[0].startswith("/")) or argv[0].startswith("-")):
            argv = argv[1:]
    return argv


def _exec_binary(argv: list[str]) -> str:
    if not argv:
        return ""
    prog = argv[0]
    if (os.path.basename(prog) in ("sh", "bash", "python3", "python") and len(argv) > 1
            and not argv[1].startswith("-")):
        prog = argv[1]  # `python3 /path/app.py` → the script is the program
    if os.path.isabs(prog):
        return prog
    return shutil.which(prog) or ""


def _classify_local(argv: list[str], binary: str) -> tuple[str, str]:
    """(kind, how-to-remove hint) for an app no package manager owns."""
    joined = " ".join(argv)
    if re.search(r"--app-id=|--app=", joined):
        browser = os.path.basename(argv[0]) if argv else "the browser"
        return "Web app", (f"Installed from a browser ({browser}). Remove it from the browser's "
                           "apps page (e.g. brave://apps or chrome://apps).")
    if re.search(r"\.appimage\b", joined, re.I):
        return "AppImage", "A self-contained AppImage file: delete the file and this launcher."
    if binary.startswith(os.path.expanduser("~") + "/"):
        return ("Your own script or program",
                "Lives in your home folder: delete its files and this launcher.")
    if binary:
        return "Manual install", ("Installed by its own installer, not a package manager. Use "
                                  "the vendor's uninstaller if it has one.")
    return "Launcher only", "Its program wasn't found — this launcher may be left over."


def scan_launchers() -> list[dict]:
    """Every visible launcher, honouring user overrides (the same desktop-id in a
    higher-precedence dir hides the lower one — exactly what the desktop menu does)."""
    found, seen = [], set()
    for d in application_dirs():
        for path in sorted(glob.glob(os.path.join(d, "*.desktop"))):
            did = os.path.basename(path)
            if did in seen:
                continue
            seen.add(did)
            entry = read_desktop(path)
            if _visible_launcher(entry):
                found.append({"id": did, "path": path, "dir": d, "entry": entry})
    return found


def _dpkg_owners(paths: list[str]) -> dict[str, list[str]]:
    """path -> [package, …] for every path dpkg knows about (one dpkg-query call)."""
    if not paths or not shutil.which("dpkg-query"):
        return {}
    _, out, _ = _run(["dpkg-query", "-S", *paths])
    owners: dict[str, list[str]] = {}
    for line in out.splitlines():
        if ": " not in line or line.startswith("diversion "):
            continue
        pkgs, path = line.rsplit(": ", 1)
        owners[path] = [p.strip().split(":")[0] for p in pkgs.split(",")]
    return owners


def attach_launchers(rows: list[dict]) -> None:
    """Map every launcher on the machine to the package that owns it; launchers nothing owns
    become 'local' rows (web apps, AppImages, manual installs, scripts)."""
    for r in rows:
        r.setdefault("icon", "")
        r.setdefault("desktop", "")
        r["package_name"] = r["name"]
        r["is_app"] = r["source"] == "flatpak"  # `flatpak list --app` is apps only

    apt_rows: dict[str, dict] = {}
    for r in rows:
        if r["source"] == "apt":
            apt_rows.setdefault(r["name"], r)
    snaps = {r["id"]: r for r in rows if r["source"] == "snap"}
    flatpaks = {r["id"]: r for r in rows if r["source"] == "flatpak"}
    flatpak_dirs = {os.path.realpath(d) for d in _FLATPAK_APPDIRS}
    app_dirs = application_dirs()

    def claim(row, launcher, score):
        name = _localized(launcher["entry"], "Name")
        if name and name not in row.setdefault("_apps", []):
            row["_apps"].append(name)
        if not row["desktop"] or score > row.get("_score", -1):
            _apply_launcher(row, launcher["path"], launcher["entry"])
            row["_score"] = score

    unowned = []
    for ln in scan_launchers():
        stem = ln["id"][:-len(".desktop")]
        if ln["dir"] in flatpak_dirs:
            fp = flatpaks.get(stem) or next((v for k, v in flatpaks.items()
                                             if stem.startswith(k + ".")), None)
            if fp:
                claim(fp, ln, 2 if fp["id"] == stem else 1)
                continue
        if ln["dir"] == os.path.realpath(_SNAP_APPDIR):
            sn = snaps.get(stem.split("_")[0])
            if sn:
                claim(sn, ln, 1)
                continue
        unowned.append(ln)

    # 1) launcher files dpkg installed itself (a user override of the same id still counts)
    def same_id_paths(ln):
        return [p for p in (os.path.join(d, ln["id"]) for d in app_dirs) if os.path.exists(p)]

    owners = _dpkg_owners(sorted({p for ln in unowned for p in same_id_paths(ln)}))
    rest = []
    for ln in unowned:
        stem = ln["id"][:-len(".desktop")]
        pkgs = next((owners[p] for p in same_id_paths(ln) if p in owners), [])
        rows_for = [apt_rows[p] for p in pkgs if p in apt_rows]
        for row in rows_for:
            claim(row, ln, 2 if stem == row["name"] else (1 if stem.startswith(row["name"]) else 0))
        if not rows_for:
            rest.append(ln)

    # 2) launchers dpkg didn't install: attribute by the program they run, else they're local apps
    for ln in rest:
        ln["argv"] = _exec_argv(ln["entry"])
        ln["binary"] = _exec_binary(ln["argv"])
    bins = sorted({p for ln in rest if ln["binary"]
                   for p in (ln["binary"], os.path.realpath(ln["binary"])) if os.path.exists(p)})
    bin_owners = _dpkg_owners(bins)
    for ln in rest:
        binary = ln["binary"]
        kind, hint = _classify_local(ln["argv"], binary)
        pkgs = bin_owners.get(binary) or bin_owners.get(os.path.realpath(binary)) or []
        row = next((apt_rows[p] for p in pkgs if p in apt_rows), None)
        if row is not None and kind != "Web app":
            if not row["desktop"]:  # a package's app whose launcher was added by hand
                claim(row, ln, 0)
            continue  # otherwise it's only a duplicate launcher for an app already listed
        target = next((a for a in ln["argv"] if a.lower().endswith(".appimage")), "")
        size_kb = os.path.getsize(target) // 1024 if target and os.path.isfile(target) else 0
        stem = ln["id"][:-len(".desktop")]
        local = {"source": "local", "id": stem, "name": stem, "version": "",
                 "size": _human_kb(size_kb), "size_kb": size_kb, "summary": "",
                 "protected": False, "icon": "", "desktop": "", "package_name": stem,
                 "kind": kind, "remove_hint": hint,
                 "location": target or binary or ln["path"], "launcher_path": ln["path"]}
        _apply_launcher(local, ln["path"], ln["entry"])
        rows.append(local)

    for r in rows:
        r.pop("_score", None)
        r["includes"] = [n for n in r.pop("_apps", []) if n != r["name"]]
        if r["source"] == "flatpak":
            r["size_kb"] = _parse_size_kb(r["size"])
        elif r["source"] == "snap":
            # removing a snap frees every stored revision
            kb = sum(os.path.getsize(p) for p in glob.glob(f"/var/lib/snapd/snaps/{r['id']}_*.snap")
                     if os.path.isfile(p)) // 1024
            r["size_kb"] = kb
            r["size"] = _human_kb(kb)
        r.setdefault("size_kb", 0)


def apt_installed_kb(pkgs: list[str]) -> int:
    """Total Installed-Size (KiB) of the given apt packages — for the 'space freed' estimate."""
    if not pkgs:
        return 0
    _, out, _ = _run(["dpkg-query", "-W", "-f", "${Installed-Size}\n", *pkgs])
    return sum(int(x) for x in out.split() if x.isdigit())


# ── removal planning (safety) ──────────────────────────────────────────────────────────────────
def simulate_apt_remove(pkg: str, purge: bool = False) -> tuple[list[str], list[str]]:
    """Return (packages_that_would_be_removed, protected_ones_in_that_set) — no changes made."""
    op = "purge" if purge else "remove"
    rc, out, err = _run(["apt-get", "-s", op, pkg])
    removed = [ln.split()[1] for ln in out.splitlines()
               if ln.startswith("Remv ") and len(ln.split()) > 1]
    if not removed and rc != 0:
        removed = []  # simulation failed (e.g. package not found); caller handles empty
    protected = [p for p in removed if is_protected(p)]
    return removed, protected


def remove_command(item: dict, purge: bool = False) -> list[str]:
    """Build the argv for uninstalling `item`. apt/snap/system-flatpak escalate via pkexec."""
    src, ident = item["source"], item["id"]
    if src == "apt":
        op = "purge" if purge else "remove"
        return ["pkexec", "apt-get", op, "-y", ident]
    if src == "snap":
        return ["pkexec", "snap", "remove", ident]
    if src == "flatpak":
        base = ["flatpak", "uninstall", "-y", ident]
        return ["pkexec", *base] if item.get("installation") == "system" else base
    raise ValueError(f"unknown source {src}")


# ── autostart ("Startup Apps" / the edit side) ──────────────────────────────────────────────────
_USER_AUTOSTART = os.path.expanduser("~/.config/autostart")
_SYS_AUTOSTART = "/etc/xdg/autostart"


def _desktop_field(path: str, key: str) -> str:
    try:
        with open(path, encoding="utf-8", errors="replace") as fh:
            for line in fh:
                if line.startswith(key + "="):
                    return line.split("=", 1)[1].strip()
    except OSError:
        pass
    return ""


def _autostart_enabled(path: str) -> bool:
    if _desktop_field(path, "Hidden").lower() == "true":
        return False
    val = _desktop_field(path, "X-GNOME-Autostart-enabled").lower()
    return val != "false"


def list_autostart() -> list[dict]:
    items, seen = [], set()
    for path in sorted(glob.glob(os.path.join(_USER_AUTOSTART, "*.desktop"))):
        base = os.path.basename(path)
        seen.add(base)
        entry = read_desktop(path)
        items.append({"name": _localized(entry, "Name") or base, "path": path,
                      "comment": _localized(entry, "Comment"), "icon": entry.get("Icon", ""),
                      "kind": "user", "enabled": _autostart_enabled(path)})
    for path in sorted(glob.glob(os.path.join(_SYS_AUTOSTART, "*.desktop"))):
        base = os.path.basename(path)
        if base in seen:  # a user override already shown above wins
            continue
        override = os.path.join(_USER_AUTOSTART, base)
        enabled = _autostart_enabled(override) if os.path.exists(override) else True
        entry = read_desktop(path)
        items.append({"name": _localized(entry, "Name") or base, "path": path,
                      "comment": _localized(entry, "Comment"), "icon": entry.get("Icon", ""),
                      "kind": "system", "enabled": enabled})
    items.sort(key=lambda i: i["name"].lower())
    return items


def set_autostart_enabled(item: dict, enabled: bool) -> None:
    """Enable/disable an autostart entry at the USER level only (no root, fully reversible)."""
    os.makedirs(_USER_AUTOSTART, exist_ok=True)
    base = os.path.basename(item["path"])
    target = os.path.join(_USER_AUTOSTART, base)
    if item["kind"] == "system" and not os.path.exists(target):
        shutil.copyfile(item["path"], target)  # copy the system entry, then toggle our copy
    lines, wrote = [], False
    with open(target, encoding="utf-8", errors="replace") as fh:
        for line in fh:
            if line.startswith("X-GNOME-Autostart-enabled=") or line.startswith("Hidden="):
                continue  # drop old state lines; we rewrite them
            lines.append(line.rstrip("\n"))
            if line.strip() == "[Desktop Entry]" and not wrote:
                lines.append(f"X-GNOME-Autostart-enabled={'true' if enabled else 'false'}")
                lines.append(f"Hidden={'false' if enabled else 'true'}")
                wrote = True
    if not wrote:  # no [Desktop Entry] header found; prepend a minimal one
        lines = ["[Desktop Entry]",
                 f"X-GNOME-Autostart-enabled={'true' if enabled else 'false'}",
                 f"Hidden={'false' if enabled else 'true'}", *lines]
    with open(target, "w", encoding="utf-8") as fh:
        fh.write("\n".join(lines) + "\n")


def remove_autostart(item: dict) -> None:
    """Remove a USER autostart entry (deletes the file). System entries are disabled, not deleted."""
    if item["kind"] == "user":
        try:
            os.remove(item["path"])
        except OSError:
            pass
    else:
        set_autostart_enabled(item, False)


# ── system monitor (read-only: /proc and /sys only — no root, no extra packages) ────────────────
_GPU_VENDOR = {"0x8086": "Intel", "0x1002": "AMD", "0x10de": "NVIDIA"}
_CPU_CHIPS = ("coretemp", "k10temp", "zenpower", "cpu_thermal")
_GPU_CHIPS = ("amdgpu", "radeon", "nouveau", "i915", "xe")
_CPU_ZONES = ("x86_pkg_temp", "cpu-thermal", "cpu_thermal", "soc_thermal")
_CHIP_LABEL = (("nvme", "NVMe SSD"), ("drivetemp", "Hard drive"), ("pch_", "Chipset"),
               ("acpitz", "Motherboard"), ("iwlwifi", "Wi-Fi card"), ("BAT", "Battery"))
_GPU_LABEL = {"edge": "GPU", "junction": "GPU hotspot", "mem": "GPU memory"}
_INTERPRETER_RE = re.compile(r"^(python[\d.]*|node|bash|sh|dash|perl|ruby|java)$")


def _read(path: str) -> str:
    try:
        with open(path, encoding="utf-8", errors="replace") as fh:
            return fh.read().strip()
    except OSError:
        return ""


def _read_int(path: str) -> int | None:
    try:
        return int(_read(path))
    except ValueError:
        return None


def _fmt_mhz(mhz: float | None) -> str:
    if not mhz:
        return "—"
    return f"{mhz / 1000:.1f} GHz" if mhz >= 1000 else f"{mhz:.0f} MHz"


def _fmt_duration(seconds: float) -> str:
    minutes = int(seconds // 60)
    d, h, m = minutes // 1440, minutes // 60 % 24, minutes % 60
    if d:
        return f"{d} day{'s' if d != 1 else ''} {h} h"
    return f"{h} h {m} min" if h else f"{m} min"


def cpu_model() -> str:
    for line in _read("/proc/cpuinfo").splitlines():
        key, _, val = line.partition(":")
        if key.strip() in ("model name", "Model", "Hardware") and val.strip():
            return re.sub(r"\s+", " ", re.sub(r"\((R|TM)\)|\bCPU\b", "", val)).strip()
    return os.uname().machine


def _cpu_times() -> list[tuple[str, int, int]]:
    """(name, busy, total) jiffies for the whole machine ("cpu") and each logical CPU."""
    out = []
    for line in _read("/proc/stat").splitlines():
        if not line.startswith("cpu"):
            break
        name, *vals = line.split()
        v = [int(x) for x in vals]
        total = sum(v[:8])  # user..steal (guest time is already inside user)
        out.append((name, total - v[3] - (v[4] if len(v) > 4 else 0), total))
    return out


def _proc_times() -> dict[int, tuple[str, int]]:
    """pid -> (comm, utime+stime jiffies) for every process."""
    out = {}
    for pid in os.listdir("/proc"):
        if not pid.isdigit():
            continue
        stat = _read(f"/proc/{pid}/stat")
        close = stat.rfind(")")  # comm may itself contain spaces or ')'
        fields = stat[close + 2:].split()
        if close < 0 or len(fields) < 13:
            continue
        out[int(pid)] = (stat[stat.find("(") + 1:close], int(fields[11]) + int(fields[12]))
    return out


def _proc_name(pid: int, comm: str) -> str:
    """A readable program name: argv[0]'s basename, or the script an interpreter is running."""
    argv = [a for a in _read(f"/proc/{pid}/cmdline").split("\0") if a]
    if not argv:
        return re.sub(r"[/:].*", "", comm)  # kernel thread: kworker/3:2-i915 -> kworker
    first = argv[0] if os.path.exists(argv[0]) else argv[0].split(" ")[0]
    name = os.path.basename(first) or comm
    if _INTERPRETER_RE.match(name):  # python3 app.py -> app.py, node server.js -> server.js
        for prev, arg in zip(argv, argv[1:]):
            if prev in ("-c", "-e", "--eval"):
                break
            if prev == "-m":
                return arg
            if not arg.startswith("-"):
                return os.path.basename(arg)
    return name


def _proc_memory_kb(pid: int, page_kb: int) -> int:
    """Proportional set size — shared pages split between the processes using them, so a
    many-process app (browsers) isn't counted several times. Falls back to resident size for
    processes whose smaps we may not read (other users')."""
    for line in _read(f"/proc/{pid}/smaps_rollup").splitlines():
        if line.startswith("Pss:"):
            return int(line.split()[1])
    statm = _read(f"/proc/{pid}/statm").split()
    return int(statm[1]) * page_kb if len(statm) > 1 else 0


def _meminfo() -> dict:
    vals = {}
    for line in _read("/proc/meminfo").splitlines():
        key, _, rest = line.partition(":")
        if rest.split():
            vals[key] = int(rest.split()[0])
    total = vals.get("MemTotal", 0)
    swap = vals.get("SwapTotal", 0)
    return {"total_kb": total, "used_kb": total - vals.get("MemAvailable", vals.get("MemFree", 0)),
            "swap_total_kb": swap, "swap_used_kb": swap - vals.get("SwapFree", 0)}


def _disks() -> list[dict]:
    out = []
    for mount, name in (("/", "System"), ("/home", "Home")):
        if mount != "/" and not os.path.ismount(mount):
            continue
        try:
            st = os.statvfs(mount)
        except OSError:
            continue
        used = (st.f_blocks - st.f_bfree) * st.f_frsize // 1024
        free = st.f_bavail * st.f_frsize // 1024  # what you can actually still write
        out.append({"mount": mount, "name": name, "used_kb": used, "free_kb": free})
    return out


def _hwmon_temps() -> list[dict]:
    """Every temperature sensor the kernel exposes under /sys/class/hwmon."""
    def index(path):
        m = re.search(r"temp(\d+)_input$", path)
        return int(m.group(1)) if m else 0

    out = []
    for hw in sorted(glob.glob("/sys/class/hwmon/hwmon*")):
        chip = _read(os.path.join(hw, "name"))
        device = os.path.realpath(os.path.join(hw, "device"))
        for inp in sorted(glob.glob(os.path.join(hw, "temp*_input")), key=index):
            milli = _read_int(inp)
            if milli is None or not 0 < milli < 127_000:
                continue  # unreadable, or a "no sensor here" sentinel value
            base = inp[:-len("_input")]
            crit = _read_int(base + "_crit") or _read_int(base + "_max")
            out.append({"chip": chip, "label": _read(base + "_label"), "c": milli / 1000,
                        "crit": crit / 1000 if crit and crit > 0 else None, "device": device})
    return out


def _friendly_temps(raw: list[dict]) -> list[dict]:
    """CPU package + cores, GPU sensors, then one line (the hottest reading) per other device."""
    cpu, gpu, other = [], [], {}
    for t in raw:
        chip, lab = t["chip"], t["label"]
        if chip in _CPU_CHIPS:
            name = ("CPU package" if lab.startswith("Package") else
                    lab if lab.startswith("Core") else f"CPU ({lab})" if lab else "CPU")
            cpu.append({"kind": "cpu", "label": name, "c": t["c"], "crit": t["crit"]})
        elif chip in _GPU_CHIPS:
            name = _GPU_LABEL.get(lab, f"GPU {lab}".strip())
            gpu.append({"kind": "gpu", "label": name, "c": t["c"], "crit": t["crit"]})
        else:
            name = next((n for prefix, n in _CHIP_LABEL if chip.startswith(prefix)), chip)
            if name not in other or t["c"] > other[name]["c"]:
                other[name] = {"kind": "other", "label": name, "c": t["c"], "crit": t["crit"]}
    if not cpu:  # no hwmon CPU driver (some ARM boards): fall back to the thermal zones
        for zone in sorted(glob.glob("/sys/class/thermal/thermal_zone*")):
            milli = _read_int(os.path.join(zone, "temp"))
            if _read(os.path.join(zone, "type")) in _CPU_ZONES and milli and 0 < milli < 127_000:
                cpu.append({"kind": "cpu", "label": "CPU", "c": milli / 1000, "crit": None})
                break
    cpu.sort(key=lambda t: t["label"] != "CPU package")
    return cpu + gpu + sorted(other.values(), key=lambda t: t["label"])


def list_gpus() -> list[dict]:
    """Graphics adapters (static facts — live readings come from SystemSampler.sample())."""
    gpus = []
    for card in sorted(glob.glob("/sys/class/drm/card*")):
        if not re.fullmatch(r"card\d+", os.path.basename(card)):
            continue  # connectors such as card1-eDP-1
        dev = os.path.realpath(os.path.join(card, "device"))
        driver = os.path.basename(os.path.realpath(os.path.join(dev, "driver")))
        if driver in ("simple-framebuffer", "simpledrm", "efifb"):
            continue  # boot framebuffer, not a GPU
        vendor = _GPU_VENDOR.get(_read(os.path.join(dev, "vendor")), "")
        slot = os.path.basename(dev)
        model = ""
        rc, out, _ = _run(["lspci", "-mm", "-s", slot], timeout=5)
        if rc == 0 and out.strip():
            try:
                fields = shlex.split(out.splitlines()[0])
            except ValueError:
                fields = []
            if len(fields) >= 4:  # slot, class, vendor, device — prefer the marketing name
                m = re.search(r"\[([^\]]+)\]", fields[3])
                model = m.group(1) if m else fields[3]
        gpus.append({"card": card, "device": dev, "driver": driver, "vendor": vendor,
                     "name": " ".join(x for x in (vendor, model) if x) or f"{driver} graphics",
                     "integrated": vendor == "Intel" and driver in ("i915", "xe")
                     and slot.endswith(":00:02.0")})
    gpus.sort(key=lambda g: g["integrated"])  # a dedicated card first when there is one
    return gpus


def _nvidia_smi() -> dict[str, dict]:
    """Readings for NVIDIA's proprietary driver, which exposes no hwmon sensors. Keyed by bus."""
    rc, out, _ = _run(["nvidia-smi", "--query-gpu=pci.bus_id,temperature.gpu,utilization.gpu,"
                       "clocks.gr,clocks.max.gr,memory.used,memory.total",
                       "--format=csv,noheader,nounits"], timeout=5)
    readings = {}
    for line in out.splitlines() if rc == 0 else []:
        f = [x.strip() for x in line.split(",")]
        if len(f) < 7:
            continue
        nums = []
        for x in f[1:]:
            try:
                nums.append(float(x))
            except ValueError:
                nums.append(None)  # "[N/A]"
        temp, busy, mhz, mhz_max, used, total = nums
        readings[f[0][-7:].lower()] = {
            "temp": temp, "busy": busy, "mhz": mhz, "mhz_max": mhz_max,
            "vram_used_kb": used * 1024 if used is not None else None,
            "vram_total_kb": total * 1024 if total is not None else None}
    return readings


def _gpu_reading(gpu: dict, raw_temps: list[dict], cpu_temp: dict | None,
                 smi: dict[str, dict]) -> dict:
    card, dev = gpu["card"], gpu["device"]
    r = {"name": gpu["name"], "driver": gpu["driver"], "integrated": gpu["integrated"],
         "asleep": False, "mhz": None, "mhz_max": None, "busy": None, "vram_used_kb": None,
         "vram_total_kb": None, "temp": None, "temp_crit": None, "temp_note": ""}
    if _read(os.path.join(dev, "power", "runtime_status")) == "suspended":
        r["asleep"] = True  # powered down to save energy; reading it would wake it up
        return r
    if gpu["driver"] == "i915":
        r["mhz"] = _read_int(card + "/gt_act_freq_mhz")
        r["mhz_max"] = _read_int(card + "/gt_RP0_freq_mhz") or _read_int(card + "/gt_max_freq_mhz")
    elif gpu["driver"] == "xe":
        freq = os.path.join(dev, "tile0", "gt0", "freq0")
        r["mhz"] = _read_int(freq + "/act_freq")
        r["mhz_max"] = _read_int(freq + "/rp0_freq") or _read_int(freq + "/max_freq")
    elif gpu["driver"] == "amdgpu":
        r["busy"] = _read_int(dev + "/gpu_busy_percent")
        clocks = re.findall(r"(\d+)\s*Mhz(\s*\*)?", _read(dev + "/pp_dpm_sclk"), re.I)
        if clocks:
            r["mhz_max"] = max(int(c) for c, _ in clocks)
            r["mhz"] = next((int(c) for c, star in clocks if star.strip()), None)
        used, total = _read_int(dev + "/mem_info_vram_used"), _read_int(dev + "/mem_info_vram_total")
        if used is not None and total:
            r["vram_used_kb"], r["vram_total_kb"] = used // 1024, total // 1024
    elif gpu["driver"] == "nvidia":
        reading = smi.get(os.path.basename(dev)[-7:].lower(), {})
        r.update({k: v for k, v in reading.items() if k != "temp"})
        if reading.get("temp") is not None:
            r["temp"] = reading["temp"]
    own = [t for t in raw_temps if t["device"] == dev]
    if own:  # prefer the "edge" sensor; otherwise the first one the driver lists
        best = next((t for t in own if t["label"] == "edge"), own[0])
        r["temp"], r["temp_crit"] = best["c"], best["crit"]
    elif r["temp"] is None and gpu["integrated"] and cpu_temp:
        r["temp"], r["temp_crit"] = cpu_temp["c"], cpu_temp["crit"]
        r["temp_note"] = "Built into the processor, so it shares the CPU package sensor."
    elif r["temp"] is None:
        r["temp_note"] = "This graphics driver doesn't report a temperature."
    return r


class SystemSampler:
    """Live readings for the System monitor page and `--monitor`. Usage figures are rates, so
    they come from the difference between two successive sample() calls (the first has none)."""

    def __init__(self):
        self.cpu_name = cpu_model()
        self.gpus = list_gpus()
        self._names: dict[int, tuple[str, str]] = {}  # pid -> (comm, display name)
        self._memory: dict[int, tuple[float, int]] = {}  # pid -> (read at, kB); PSS is costly
        self.reset()

    def reset(self) -> None:
        self._cpu_prev: dict[str, tuple[int, int]] = {}
        self._proc_prev: dict[int, tuple[str, int]] = {}

    def _name(self, pid: int, comm: str) -> str:
        cached = self._names.get(pid)
        if cached is None or cached[0] != comm:  # new process, or it exec'd something else
            cached = self._names[pid] = (comm, _proc_name(pid, comm))
        return cached[1]

    def sample(self, top: int = 8) -> dict:
        now, cpu_now, procs = time.time(), _cpu_times(), _proc_times()
        usage, total_delta, cores = None, 0, []
        for name, busy, total in cpu_now:
            prev = self._cpu_prev.get(name)
            pct = None
            if prev and total > prev[1]:
                pct = 100 * (busy - prev[0]) / (total - prev[1])
            if name == "cpu":
                usage, total_delta = pct, total - prev[1] if prev else 0
            else:
                khz = _read_int(f"/sys/devices/system/cpu/{name}/cpufreq/scaling_cur_freq")
                cores.append({"name": name, "usage": pct, "mhz": khz / 1000 if khz else None})
        self._cpu_prev = {name: (busy, total) for name, busy, total in cpu_now}

        groups: dict[str, dict] = {}
        for pid, (comm, ticks) in procs.items():
            name = self._name(pid, comm)
            g = groups.setdefault(name, {"name": name, "cpu": 0.0, "pids": []})
            g["pids"].append(pid)
            prev = self._proc_prev.get(pid)
            if prev and prev[0] == comm and total_delta > 0:
                g["cpu"] += 100 * (ticks - prev[1]) / total_delta  # share of the whole machine
        self._proc_prev = procs
        self._names = {pid: v for pid, v in self._names.items() if pid in procs}
        self._memory = {pid: v for pid, v in self._memory.items() if pid in procs}
        page_kb = os.sysconf("SC_PAGE_SIZE") // 1024
        processes = []
        if usage is not None:
            for g in sorted(groups.values(), key=lambda g: -g["cpu"])[:top]:
                mem = 0
                for pid in g["pids"]:  # only for the groups we show, and at most every 10 s
                    read_at, kb = self._memory.get(pid, (0.0, 0))
                    if now - read_at >= 10:
                        kb = _proc_memory_kb(pid, page_kb)
                        self._memory[pid] = (now, kb)
                    mem += kb
                processes.append({"name": g["name"], "cpu": g["cpu"], "memory_kb": mem,
                                  "count": len(g["pids"])})

        raw = _hwmon_temps()
        temps = _friendly_temps(raw)
        cpu_temp = next((t for t in temps if t["kind"] == "cpu"), None)
        smi = _nvidia_smi() if any(g["driver"] == "nvidia" for g in self.gpus) else {}
        max_khz = _read_int("/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq")
        uptime = _read("/proc/uptime").split()
        return {
            "time": now,
            "cpu": {"model": self.cpu_name, "usage": usage, "cores": cores,
                    "mhz_max": max_khz / 1000 if max_khz else None},
            "cpu_temp": cpu_temp,
            "throttle_count": _read_int(
                "/sys/devices/system/cpu/cpu0/thermal_throttle/package_throttle_count"),
            "load": os.getloadavg(),
            "uptime": float(uptime[0]) if uptime else 0.0,
            "memory": _meminfo(),
            "disks": _disks(),
            "temps": temps,
            "gpus": [_gpu_reading(g, raw, cpu_temp, smi) for g in self.gpus],
            "processes": processes,
        }


def _print_monitor(s: dict) -> None:
    cpu, mem = s["cpu"], s["memory"]
    mhz = [c["mhz"] for c in cpu["cores"] if c["mhz"]]
    print(f"CPU      {cpu['model']} — {len(cpu['cores'])} threads, {cpu['usage']:.0f}% busy, "
          f"{_fmt_mhz(sum(mhz) / len(mhz) if mhz else None)} avg "
          f"(max {_fmt_mhz(cpu['mhz_max'])})")
    if s["cpu_temp"]:
        line = f"         {s['cpu_temp']['c']:.0f} °C ({s['cpu_temp']['label']})"
        if s["throttle_count"] is not None:
            line += f" · slowed down to cool off {s['throttle_count']:,} times since boot"
        print(line)
    print(f"Load     {' '.join(f'{x:.2f}' for x in s['load'])} · up {_fmt_duration(s['uptime'])}")
    print(f"Memory   {_human_kb(mem['used_kb'])} of {_human_kb(mem['total_kb'])} used"
          + (f" · swap {_human_kb(mem['swap_used_kb'])} of {_human_kb(mem['swap_total_kb'])}"
             if mem["swap_total_kb"] else ""))
    for d in s["disks"]:
        print(f"Disk     {d['name']} ({d['mount']}): {_human_kb(d['free_kb'])} free of "
              f"{_human_kb(d['used_kb'] + d['free_kb'])}")
    for g in s["gpus"]:
        kind = "integrated" if g["integrated"] else "dedicated"
        if g["asleep"]:
            print(f"GPU      {g['name']} ({g['driver']}, {kind}) — asleep")
            continue
        parts = []
        if g["mhz_max"]:
            parts.append(f"{g['mhz'] or 0:.0f} of {g['mhz_max']:.0f} MHz")
        if g["busy"] is not None:
            parts.append(f"{g['busy']:.0f}% busy")
        if g["temp"] is not None:
            shared = " (CPU package sensor)" if g["integrated"] and g["temp_note"] else ""
            parts.append(f"{g['temp']:.0f} °C{shared}")
        print(f"GPU      {g['name']} ({g['driver']}, {kind}) — "
              + (" · ".join(parts) or g["temp_note"]))
    print("Temps    " + ", ".join(f"{t['label']} {t['c']:.0f} °C" for t in s["temps"]))
    print("Top      " + ", ".join(f"{p['name']} {p['cpu']:.1f}%" for p in s["processes"][:5]))


# ── CLI (headless verification / scripting) ─────────────────────────────────────────────────────
def _cli(argv: list[str]) -> int:
    if "--monitor" in argv:
        sampler = SystemSampler()
        sampler.sample()
        time.sleep(1)  # usage figures are rates: they need two samples
        snap = sampler.sample()
        if "--json" in argv:
            print(json.dumps(snap, indent=2))
        else:
            _print_monitor(snap)
        return 0
    if "--check" in argv:
        i = argv.index("--check")
        if i + 1 >= len(argv):
            print("usage: --check <apt-package>")
            return 2
        pkg = argv[i + 1]
        removed, protected = simulate_apt_remove(pkg)
        if not removed:
            print(f"{pkg}: not installed, or nothing to remove.")
            return 1
        print(f"Removing '{pkg}' via apt would remove {len(removed)} package(s):")
        for p in removed:
            print(f"  - {p}" + ("   [PROTECTED]" if is_protected(p) else ""))
        if protected:
            print(f"\nBLOCKED: this would remove essential package(s): {', '.join(protected)}")
            return 3
        print("\nOK: no essential packages in the removal set.")
        return 0
    rows = list_all()
    if "--apps" in argv:
        rows = [r for r in rows if r.get("is_app")]
    if "--json" in argv:
        print(json.dumps(rows, indent=2))
    else:
        print(f"{'SOURCE':7} {'SIZE':>9}  NAME")
        for r in rows:
            flag = " *protected" if r["protected"] else ""
            print(f"{r['source']:7} {r['size']:>9}  {r['name']}{flag}")
        n = collections.Counter(r["source"] for r in rows)
        print(f"\n{len(rows)} program(s): {n['apt']} apt, {n['snap']} snap, "
              f"{n['flatpak']} flatpak, {n['local']} other (no package manager)")
    return 0


# ── GUI ─────────────────────────────────────────────────────────────────────────────────────────
_SOURCE_LABEL = {"apt": "APT", "snap": "Snap", "flatpak": "Flatpak", "local": "Other"}

# sidebar: (key, title, symbolic icon, section, page description)
_VIEWS = [
    ("apps", "Applications", "view-app-grid-symbolic", "Library",
     "Everything with a launcher on this computer, from every source."),
    ("apt", "Debian packages", "package-x-generic-symbolic", "Sources",
     "Apps installed from .deb packages with APT."),
    ("flatpak", "Flatpak", "application-x-addon-symbolic", "Sources",
     "Sandboxed apps installed with Flatpak."),
    ("snap", "Snap", "applications-utilities-symbolic", "Sources",
     "Apps installed with Snap."),
    ("local", "Unmanaged", "web-browser-symbolic", "Sources",
     "Apps no package manager tracks — web apps, AppImages, manual installs. Shown read-only."),
    ("all", "All packages", "view-list-symbolic", "Advanced",
     "Every installed package, including libraries and command-line tools."),
    ("startup", "Startup apps", "system-run-symbolic", "System",
     "Choose what starts when you log in. Turning something off is safe and reversible — "
     "system entries are only disabled for your account, never deleted."),
    ("monitor", "System monitor", "utilities-system-monitor-symbolic", "System",
     "Live readings from this computer — processor, memory, temperatures and graphics — "
     "refreshed every 2 seconds while this page is open. Read-only: nothing here needs your "
     "password."),
]

_MON_INTERVAL = 2   # seconds between samples while the System monitor page is visible
_MON_HISTORY = 60   # samples kept for the history charts (60 × 2 s = 2 minutes)

# Chart and meter colours: the dataviz reference palette's first two categorical slots
# (validated for colour-blind separation and 3:1 contrast, separate light/dark steps) plus its
# fixed status colours — those only ever appear next to a warning icon and a word.
_VIZ = {"light": {"cpu": "#2a78d6", "temp": "#eb6834"},
        "dark": {"cpu": "#3987e5", "temp": "#d95926"}}
_STATUS = {"warn": "#fab219", "hot": "#d03b3b"}
_METER_CSS = """
levelbar.meter trough { min-height: 8px; padding: 0; border: none; border-radius: 999px;
                        background-image: none; box-shadow: none;
                        background-color: alpha(%(accent)s, 0.18); }
levelbar.meter block { min-height: 8px; border: none; border-radius: 999px;
                       background-image: none; box-shadow: none; }
levelbar.meter block.filled { background-color: %(accent)s; }
levelbar.meter block.empty { background-color: transparent; }
levelbar.meter-warn trough { background-color: alpha(%(warn)s, 0.22); }
levelbar.meter-warn block.filled { background-color: %(warn)s; }
levelbar.meter-hot trough { background-color: alpha(%(hot)s, 0.18); }
levelbar.meter-hot block.filled { background-color: %(hot)s; }
.status-warn { color: %(warn)s; -gtk-icon-palette: warning %(warn)s, error %(warn)s; }
.status-hot { color: %(hot)s; -gtk-icon-palette: warning %(hot)s, error %(hot)s; }
"""

_CSS = b"""
.sidebar { background-color: mix(@theme_bg_color, @theme_fg_color, 0.035);
           border-right: 1px solid alpha(@theme_fg_color, 0.08); }
.sidebar list { background-color: transparent; }
.sidebar row { border-radius: 8px; margin: 1px 10px; }
/* themes (Zorin included) paint selection with background-image, which would cover our tint
   and (with a light accent) hide the text; reset it and keep the normal text colour */
.sidebar row:hover { background-image: none; background-color: alpha(@theme_fg_color, 0.05); }
.sidebar row:selected, .sidebar row:selected:hover {
    background-image: none; background-color: alpha(@theme_selected_bg_color, 0.24);
    color: @theme_fg_color; }
.sidebar row:selected label, .sidebar row:selected image { color: @theme_fg_color; }
.sidebar-heading { font-size: 0.78em; font-weight: bold; opacity: 0.5; }
.nav-count { font-size: 0.85em; opacity: 0.55; }
.machine-card { border-radius: 10px; padding: 12px;
                background-color: alpha(@theme_fg_color, 0.05); }

.page-title { font-size: 1.75em; font-weight: 800; }
.card { background-color: @theme_base_color; border-radius: 12px;
        border: 1px solid alpha(@theme_fg_color, 0.09); }
.card list { background-color: transparent; }
.app-list row { border-radius: 8px; margin: 1px 6px; }
.app-list row:hover { background-image: none; background-color: alpha(@theme_fg_color, 0.045); }
.app-list row:selected, .app-list row:selected:hover {
    background-image: none; background-color: alpha(@theme_selected_bg_color, 0.22);
    color: @theme_fg_color; }
.app-list row:selected label, .app-list row:selected image { color: @theme_fg_color; }

.badge { border-radius: 999px; padding: 1px 9px; font-size: 0.8em; font-weight: bold; }
.badge-apt { color: #e66100; background-color: alpha(#e66100, 0.15); }
.badge-snap { color: #2ec27e; background-color: alpha(#2ec27e, 0.15); }
.badge-flatpak { color: #3584e4; background-color: alpha(#3584e4, 0.15); }
.badge-local { color: #c061cb; background-color: alpha(#c061cb, 0.15); }
.chip { border-radius: 999px; padding: 1px 9px; font-size: 0.8em;
        background-color: alpha(@theme_fg_color, 0.08); }
.chip-protected { color: #e5a50a; background-color: alpha(#e5a50a, 0.15); }

.detail-title { font-size: 1.45em; font-weight: 800; }
.dialog-title { font-size: 1.25em; font-weight: 800; }
.props { border-radius: 10px; background-color: alpha(@theme_fg_color, 0.04); }
.prop-key { opacity: 0.6; }
.pill { border-radius: 999px; padding-left: 16px; padding-right: 16px; }
.warning-box { border-radius: 10px; padding: 10px; background-color: alpha(#e5a50a, 0.15); }
.info-box { border-radius: 10px; padding: 10px; background-color: alpha(#3584e4, 0.14); }
.pkg-list { border-radius: 8px; border: 1px solid alpha(@theme_fg_color, 0.15); }

.dim { opacity: 0.62; }
.app-title { font-weight: bold; }
.tile-value { font-size: 1.9em; font-weight: 600; }
.mon-flow flowboxchild { padding: 0; }
.tnum { font-feature-settings: "tnum"; }
.empty-title { font-size: 1.25em; font-weight: bold; }

.toast { border-radius: 999px; padding: 6px 8px 6px 18px; color: #ffffff;
         background-color: rgba(28, 28, 30, 0.96);
         box-shadow: 0 3px 10px rgba(0, 0, 0, 0.35); }
.toast label { color: #ffffff; }
.toast button { color: #ffffff; }
"""


def _run_streaming(argv: list[str], on_line, timeout: int = 900) -> tuple[int, str]:
    """Run argv (never a shell), calling on_line(text) per output line. Returns (rc, tail)."""
    try:
        p = subprocess.Popen(argv, stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
                             text=True, bufsize=1)
    except FileNotFoundError:
        return 127, f"{argv[0]}: not found"
    killer = threading.Timer(timeout, p.kill)
    killer.start()
    tail: list[str] = []
    try:
        for line in p.stdout:
            line = line.rstrip()
            if line:
                tail = (tail + [line])[-40:]
                on_line(line)
        rc = p.wait()
    finally:
        killer.cancel()
    return rc, "\n".join(tail)


def _machine_info() -> dict:
    pretty = ""
    try:
        with open("/etc/os-release", encoding="utf-8") as fh:
            for line in fh:
                if line.startswith("PRETTY_NAME="):
                    pretty = line.split("=", 1)[1].strip().strip('"')
    except OSError:
        pass
    return {"host": os.uname().nodename, "os": pretty or os.uname().sysname,
            "kernel": os.uname().release}


def run_gui() -> int:
    import gi
    # pin every versioned namespace: with GTK 4 also installed, an unpinned Gdk resolves to 4.0
    # and GTK 3 then fails to load ("Requiring namespace 'Gdk' version '3.0', but '4.0' is loaded")
    gi.require_version("Gtk", "3.0")
    gi.require_version("Gdk", "3.0")
    gi.require_version("Pango", "1.0")
    gi.require_version("PangoCairo", "1.0")
    from gi.repository import Gdk, Gio, GLib, Gtk, Pango, PangoCairo

    provider = Gtk.CssProvider()
    provider.load_from_data(_CSS)
    Gtk.StyleContext.add_provider_for_screen(Gdk.Screen.get_default(), provider,
                                             Gtk.STYLE_PROVIDER_PRIORITY_APPLICATION)

    def css(widget, *classes):
        ctx = widget.get_style_context()
        for c in classes:
            ctx.add_class(c)
        return widget

    def label(text="", *classes, xalign=0.0, wrap=False, ellipsize=True, **kw):
        lbl = Gtk.Label(label=text, xalign=xalign, **kw)
        if wrap:
            lbl.set_line_wrap(True)
            lbl.set_line_wrap_mode(Pango.WrapMode.WORD_CHAR)
        elif ellipsize:
            lbl.set_ellipsize(Pango.EllipsizeMode.END)
        return css(lbl, *classes)

    def gicon_for(icon: str, fallback: str):
        if icon and os.path.isabs(icon):
            if os.path.exists(icon):
                return Gio.FileIcon.new(Gio.File.new_for_path(icon))
            return Gio.ThemedIcon.new(fallback)
        name = re.sub(r"\.(png|svg|xpm)$", "", icon or "")
        # resolve explicitly: GTK searches *all* names in the current theme before hicolor, so a
        # themed generic fallback would otherwise win over an app icon that only lives in hicolor
        theme = Gtk.IconTheme.get_default()
        return Gio.ThemedIcon.new(name if name and theme.has_icon(name) else fallback)

    def fallback_icon(item: dict) -> str:
        return "application-x-executable" if item.get("is_app") else "package-x-generic"

    def icon_image(icon: str, fallback: str, size: int):
        img = Gtk.Image.new_from_gicon(gicon_for(icon, fallback), Gtk.IconSize.DIALOG)
        img.set_pixel_size(size)
        return img

    def symbolic(name: str, size: int = 16):
        img = Gtk.Image.new_from_icon_name(name, Gtk.IconSize.MENU)
        img.set_pixel_size(size)
        return img

    def badge(source: str):
        return label(_SOURCE_LABEL.get(source, source), "badge", f"badge-{source}",
                     ellipsize=False, valign=Gtk.Align.CENTER)

    def chip(text: str, *classes):
        return label(text, "chip", *classes, ellipsize=False, valign=Gtk.Align.CENTER)

    def pill_button(text: str, *classes, icon: str = ""):
        btn = Gtk.Button()
        box = Gtk.Box(spacing=6, halign=Gtk.Align.CENTER)
        if icon:
            box.pack_start(symbolic(icon), False, False, 0)
        box.pack_start(Gtk.Label(label=text), False, False, 0)
        btn.add(box)
        return css(btn, "pill", *classes)

    def scrolled(child, *classes):
        sw = Gtk.ScrolledWindow(hscrollbar_policy=Gtk.PolicyType.NEVER)
        sw.add(child)
        return css(sw, *classes)

    class NavRow(Gtk.ListBoxRow):
        def __init__(self, key, title, icon, section):
            super().__init__()
            self.key, self.section = key, section
            box = Gtk.Box(spacing=10, margin_top=7, margin_bottom=7,
                          margin_start=10, margin_end=10)
            box.pack_start(symbolic(icon), False, False, 0)
            box.pack_start(label(title), True, True, 0)
            self.count = label("", "nav-count", xalign=1.0, ellipsize=False)
            box.pack_start(self.count, False, False, 0)
            self.add(box)

    class ProgramRow(Gtk.ListBoxRow):
        def __init__(self, item: dict):
            super().__init__()
            self.item = item
            box = Gtk.Box(spacing=12, margin_top=8, margin_bottom=8,
                          margin_start=10, margin_end=12)
            box.pack_start(icon_image(item.get("icon", ""), fallback_icon(item), 36),
                           False, False, 0)
            text = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=2,
                           valign=Gtk.Align.CENTER)
            text.pack_start(label(item["name"], "app-title"), False, False, 0)
            sub = item.get("summary") or item["id"]
            if item["source"] == "local":  # desktop-ids like brave-<hash>-Default mean nothing
                sub = "  ·  ".join(x for x in (item.get("summary"), item.get("kind")) if x)
            elif item.get("summary") and item["name"] != item["id"]:
                sub = f"{item['summary']}  ·  {item['id']}"
            text.pack_start(label(sub, "dim"), False, False, 0)
            box.pack_start(text, True, True, 0)
            if item["protected"]:
                lock = symbolic("changes-prevent-symbolic")
                lock.set_tooltip_text("Protected system component — can't be uninstalled here")
                box.pack_start(css(lock, "dim"), False, False, 0)
            box.pack_start(badge(item["source"]), False, False, 0)
            size = item["size"] if item["size"] not in ("-", "") else ""
            box.pack_start(label(size, "dim", xalign=1.0, width_chars=9, ellipsize=False),
                           False, False, 0)
            self.add(box)

    class StartupRow(Gtk.ListBoxRow):
        def __init__(self, item: dict, on_switch, on_delete):
            super().__init__(activatable=False, selectable=False)
            self.item = item
            box = Gtk.Box(spacing=12, margin_top=9, margin_bottom=9,
                          margin_start=10, margin_end=12)
            box.pack_start(icon_image(item.get("icon", ""), "application-x-executable", 36),
                           False, False, 0)
            text = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=2,
                           valign=Gtk.Align.CENTER)
            text.pack_start(label(item["name"], "app-title"), False, False, 0)
            text.pack_start(label(item.get("comment") or os.path.basename(item["path"]), "dim"),
                            False, False, 0)
            box.pack_start(text, True, True, 0)
            box.pack_start(chip("System" if item["kind"] == "system" else "Added by you"),
                           False, False, 0)
            if item["kind"] == "user":
                rm = Gtk.Button.new_from_icon_name("user-trash-symbolic", Gtk.IconSize.BUTTON)
                rm.set_tooltip_text("Delete this startup entry")
                rm.set_relief(Gtk.ReliefStyle.NONE)
                rm.set_valign(Gtk.Align.CENTER)
                rm.connect("clicked", lambda *_: on_delete(item))
                box.pack_start(rm, False, False, 0)
            self.switch = Gtk.Switch(active=item["enabled"], valign=Gtk.Align.CENTER)
            self.switch.set_tooltip_text("Run at login")
            self.switch.connect("notify::active", lambda sw, _p: on_switch(self, sw.get_active()))
            box.pack_start(self.switch, False, False, 0)
            self.add(box)

    # ── system monitor widgets ───────────────────────────────────────────────────────────────
    def is_dark(widget) -> bool:
        fg = widget.get_style_context().get_color(Gtk.StateFlags.NORMAL)
        return 0.2126 * fg.red + 0.7152 * fg.green + 0.0722 * fg.blue > 0.5  # light text

    def temp_level(c, crit):
        frac = c / (crit or 100)
        return ("hot", "Hot") if frac >= 0.9 else ("warn", "Warm") if frac >= 0.8 else (None, "")

    def fill_level(frac):
        return (("hot", "Almost full") if frac >= 0.95 else
                ("warn", "Getting full") if frac >= 0.85 else (None, ""))

    def meter(**kw):
        bar = css(Gtk.LevelBar(min_value=0.0, max_value=1.0, valign=Gtk.Align.CENTER, **kw),
                  "meter")
        for offset in (Gtk.LEVEL_BAR_OFFSET_LOW, Gtk.LEVEL_BAR_OFFSET_HIGH,
                       Gtk.LEVEL_BAR_OFFSET_FULL):
            bar.remove_offset_value(offset)  # the theme would colour those; severity is ours
        return bar

    def set_meter(bar, fraction, level=None):
        bar.set_value(min(max(fraction or 0.0, 0.0), 1.0))
        ctx = bar.get_style_context()
        for lv in ("warn", "hot"):
            (ctx.add_class if lv == level else ctx.remove_class)(f"meter-{lv}")

    def status_icon():
        icon = symbolic("dialog-warning-symbolic")
        icon.set_no_show_all(True)
        return icon

    def set_status(icon, level, word=""):
        ctx = icon.get_style_context()
        for lv in ("warn", "hot"):
            (ctx.add_class if lv == level else ctx.remove_class)(f"status-{lv}")
        icon.set_tooltip_text(word or None)
        icon.set_visible(level is not None)

    class MeterRow(Gtk.Box):
        """name · meter · value. The fill colour carries severity, always together with a warning
        icon and a word ("Warm", "Hot", "Getting full"), so colour never carries it alone."""

        def __init__(self, name, groups=None):
            super().__init__(spacing=8)
            name_label = label(name, "dim", width_chars=9, max_width_chars=12)
            name_label.set_tooltip_text(name)
            self.pack_start(name_label, False, False, 0)
            self.bar = meter(hexpand=True, margin_end=2)
            self.pack_start(self.bar, True, True, 0)
            tail = Gtk.Box(spacing=8)
            self.icon = status_icon()
            tail.pack_start(self.icon, False, False, 0)
            self.word = label("", ellipsize=False)  # the status word is never cut short
            self.word.set_no_show_all(True)
            tail.pack_start(self.word, False, False, 0)
            # the number may shorten itself in a very narrow window; that keeps this row from
            # setting the whole window's minimum width
            self.value = label("…", "tnum", xalign=1.0, width_chars=6)
            tail.pack_start(self.value, True, True, 0)
            self.pack_start(tail, False, False, 0)
            if groups:  # rows sharing size groups get bars of one length, so they compare
                groups[0].add_widget(name_label)
                groups[1].add_widget(tail)

        def set(self, fraction, text, level=None, word=""):
            set_meter(self.bar, fraction, level)
            set_status(self.icon, level, word)
            self.word.set_text(word if level else "")
            self.word.set_visible(level is not None)
            self.value.set_text(text)
            self.value.set_tooltip_text(text)

    class StatTile(Gtk.Box):
        """One headline number: label · value (+ status icon and word) · context · meter."""

        def __init__(self, title):
            super().__init__(orientation=Gtk.Orientation.VERTICAL)
            css(self, "card")
            inner = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=4, margin=16)
            self.title = label(title, "dim")
            inner.pack_start(self.title, False, False, 0)
            row = Gtk.Box(spacing=8)
            self.value = label("…", "tile-value", ellipsize=False)
            row.pack_start(self.value, False, False, 0)
            self.icon = status_icon()
            row.pack_start(self.icon, False, False, 0)
            self.word = label("", "dim", ellipsize=False)
            row.pack_start(self.word, False, False, 0)
            inner.pack_start(row, False, False, 0)
            self.sub = label("", "dim")
            inner.pack_start(self.sub, False, False, 0)
            self.bar = meter(margin_top=6)
            inner.pack_start(self.bar, False, False, 0)
            self.add(inner)

        def set(self, value, sub, fraction, level=None, word="", title=None):
            if title:
                self.title.set_text(title)
            self.value.set_text(value)
            self.sub.set_text(sub)
            self.sub.set_tooltip_text(sub)
            set_meter(self.bar, fraction, level)
            set_status(self.icon, level, word)
            self.word.set_text(word if level else "")

    class HistoryChart(Gtk.DrawingArea):
        """One series over the last two minutes, newest on the right: a 2px line over a ~10% wash
        on hairline gridlines. Hovering shows a crosshair and reports that sample via on_hover."""

        def __init__(self, history, key, lo, hi, ticks, unit, series, on_hover):
            super().__init__(height_request=130, hexpand=True)
            self.history, self.key, self.series, self.on_hover = history, key, series, on_hover
            self.lo, self.hi, self.ticks, self.unit = lo, hi, ticks, unit
            self.hover = None
            self.add_events(Gdk.EventMask.POINTER_MOTION_MASK | Gdk.EventMask.LEAVE_NOTIFY_MASK)
            self.connect("draw", self._draw)
            self.connect("motion-notify-event", self._on_motion)
            self.connect("leave-notify-event", self._on_leave)

        def _box(self):  # plot area: left, right, top, bottom (leaves room for axis labels)
            return 42, self.get_allocated_width() - 8, 8, self.get_allocated_height() - 22

        def _y(self, value):
            _x0, _x1, y0, y1 = self._box()
            value = min(max(value, self.lo), self.hi)
            return y1 - (value - self.lo) / (self.hi - self.lo) * (y1 - y0)

        def _points(self):
            x0, x1, _y0, _y1 = self._box()
            step = (x1 - x0) / (_MON_HISTORY - 1)
            first = _MON_HISTORY - len(self.history)
            return [(x0 + (first + i) * step, self._y(s[self.key]), s)
                    for i, s in enumerate(self.history) if s[self.key] is not None]

        def _text(self, cr, text, x, y, fg, align):
            layout = self.create_pango_layout("")
            layout.set_markup(f"<small>{GLib.markup_escape_text(text)}</small>", -1)
            _ink, logical = layout.get_pixel_extents()
            cr.set_source_rgba(fg.red, fg.green, fg.blue, 0.6)  # muted ink, never series colour
            cr.move_to(x - logical.width * align, y - logical.height / 2)
            PangoCairo.show_layout(cr, layout)

        def _draw(self, _w, cr):
            ctx = self.get_style_context()
            fg = ctx.get_color(Gtk.StateFlags.NORMAL)
            color = Gdk.RGBA()
            color.parse(_VIZ["dark" if is_dark(self) else "light"][self.series])
            x0, x1, y0, y1 = self._box()
            cr.set_line_width(1)
            for tick in self.ticks:  # hairline grid; the baseline a step stronger
                y = round(self._y(tick)) + 0.5
                cr.set_source_rgba(fg.red, fg.green, fg.blue, 0.24 if tick == self.lo else 0.09)
                cr.move_to(x0, y)
                cr.line_to(x1, y)
                cr.stroke()
                self._text(cr, f"{tick}{self.unit}", x0 - 6, y, fg, 1.0)
            minutes = _MON_HISTORY * _MON_INTERVAL // 60
            self._text(cr, f"{minutes} min ago", x0, y1 + 12, fg, 0.0)
            self._text(cr, "now", x1, y1 + 12, fg, 1.0)

            pts = self._points()
            if not pts:
                return False
            cr.move_to(pts[0][0], y1)
            for x, y, _s in pts:
                cr.line_to(x, y)
            cr.line_to(pts[-1][0], y1)
            cr.close_path()
            cr.set_source_rgba(color.red, color.green, color.blue, 0.10)
            cr.fill()
            cr.set_line_width(2)
            cr.set_line_join(1)  # cairo.LINE_JOIN_ROUND
            cr.set_line_cap(1)   # cairo.LINE_CAP_ROUND
            cr.move_to(pts[0][0], pts[0][1])
            for x, y, _s in pts[1:]:
                cr.line_to(x, y)
            cr.set_source_rgba(color.red, color.green, color.blue, 1.0)
            cr.stroke()

            near = self._nearest(pts)
            if near is not None:  # crosshair + 8px marker with a 2px surface ring
                x, y, _s = near
                cr.set_line_width(1)
                cr.set_source_rgba(fg.red, fg.green, fg.blue, 0.35)
                cr.move_to(round(x) + 0.5, y0)
                cr.line_to(round(x) + 0.5, y1)
                cr.stroke()
                found, base = ctx.lookup_color("theme_base_color")
                if found:
                    cr.set_source_rgba(base.red, base.green, base.blue, 1.0)
                    cr.arc(x, y, 6, 0, 6.2832)
                    cr.fill()
                cr.set_source_rgba(color.red, color.green, color.blue, 1.0)
                cr.arc(x, y, 4, 0, 6.2832)
                cr.fill()
            return False

        def _nearest(self, pts):
            if self.hover is None or not pts:
                return None
            return min(pts, key=lambda p: abs(p[0] - self.hover))

        def _on_motion(self, _w, event):
            self.hover = event.x
            near = self._nearest(self._points())
            self.on_hover(near[2] if near else None)
            self.queue_draw()

        def _on_leave(self, *_):
            self.hover = None
            self.on_hover(None)
            self.queue_draw()

        def refresh(self):  # new data arrived: redraw, and re-report what's under the pointer
            if self.hover is not None:
                near = self._nearest(self._points())
                self.on_hover(near[2] if near else None)
            self.queue_draw()

    class AppManager(Gtk.Window):
        _PAGE = 150

        def __init__(self):
            super().__init__(title="App Manager")
            self.set_default_size(1260, 760)
            self.set_size_request(900, 560)
            self.set_icon_name("system-software-install")
            self._rows: list[dict] = []
            self._matches: list[dict] = []
            self._row_cache: dict = {}
            self._shown = 0
            self._rendering = False
            self._view = "apps"
            self._au: list[dict] = []
            self._current: dict | None = None
            self._busy = False
            self._toast_token = 0
            self._syncing = False
            self._views = {v[0]: v for v in _VIEWS}
            self._sampler: SystemSampler | None = None  # created on first visit (runs lspci)
            self._mon_hist = collections.deque(maxlen=_MON_HISTORY)
            self._mon_timer = None
            self._mon_busy = False
            self._mon_reset = False
            self._mon_throttle0 = None
            self._mon_hover: dict = {}
            self._mon_rows: dict = {}
            self._gpu_widgets = None
            self._viz_provider = Gtk.CssProvider()
            Gtk.StyleContext.add_provider_for_screen(Gdk.Screen.get_default(),
                                                     self._viz_provider,
                                                     Gtk.STYLE_PROVIDER_PRIORITY_APPLICATION)
            self._load_viz_css()
            Gtk.Settings.get_default().connect(
                "notify::gtk-theme-name", lambda *_: GLib.idle_add(self._load_viz_css))

            header = Gtk.HeaderBar(show_close_button=True, title="App Manager")
            menu = Gtk.MenuButton(image=symbolic("open-menu-symbolic"))
            menu.set_tooltip_text("Menu")
            menu.set_popover(self._menu_popover(menu))
            header.pack_end(menu)
            self.refresh_btn = Gtk.Button(image=symbolic("view-refresh-symbolic"))
            self.refresh_btn.set_tooltip_text("Rescan (F5)")
            self.refresh_btn.connect("clicked", lambda *_: self.refresh_current())
            header.pack_end(self.refresh_btn)
            self.set_titlebar(header)

            self.stack = Gtk.Stack(transition_type=Gtk.StackTransitionType.CROSSFADE,
                                   transition_duration=120)
            self.stack.add_named(self._programs_page(), "programs")
            self.stack.add_named(self._startup_page(), "startup")
            self.stack.add_named(self._monitor_page(), "monitor")

            body = Gtk.Box()
            body.pack_start(self._sidebar(), False, False, 0)
            body.pack_start(self.stack, True, True, 0)
            overlay = Gtk.Overlay()
            overlay.add(body)
            overlay.add_overlay(self._toast())
            self.add(overlay)

            self.connect("key-press-event", self._on_key)
            self.connect("destroy", Gtk.main_quit)
            self.refresh_programs()
            self.refresh_startup()

        # ── chrome: menu, sidebar, toast ─────────────────────────────────────────────────
        def _menu_popover(self, relative):
            pop = Gtk.Popover(relative_to=relative)
            box = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, margin=8)
            for text, cb in (("Rescan computer", self.refresh_current),
                             ("Keyboard shortcuts", self._show_shortcuts),
                             ("About App Manager", self._show_about)):
                b = Gtk.ModelButton(text=text)
                b.connect("clicked", lambda _b, f=cb: (pop.popdown(), f()))
                box.pack_start(b, False, False, 0)
            box.show_all()
            pop.add(box)
            return pop

        def _sidebar(self):
            side = css(Gtk.Box(orientation=Gtk.Orientation.VERTICAL, width_request=236),
                       "sidebar")
            self.nav = Gtk.ListBox(selection_mode=Gtk.SelectionMode.SINGLE, margin_top=6)
            self.nav.set_header_func(self._nav_header, None)
            self.nav_rows: dict[str, NavRow] = {}
            for key, title, icon, section, _desc in _VIEWS:
                row = NavRow(key, title, icon, section)
                self.nav_rows[key] = row
                self.nav.add(row)
            self.nav.connect("row-selected", self._on_nav)
            side.pack_start(scrolled(self.nav), True, True, 0)

            info = _machine_info()
            card = css(Gtk.Box(spacing=10, margin=12), "machine-card")
            card.pack_start(symbolic("computer-symbolic", 28), False, False, 0)
            txt = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=1)
            txt.pack_start(label(info["host"], "app-title"), False, False, 0)
            txt.pack_start(label(info["os"], "dim"), False, False, 0)
            self.scan_label = label(f"Linux {info['kernel']}", "dim")
            txt.pack_start(self.scan_label, False, False, 0)
            card.pack_start(txt, True, True, 0)
            card.set_tooltip_text(f"{info['host']}\n{info['os']}\nKernel {info['kernel']}")
            side.pack_end(card, False, False, 0)
            return side

        def _nav_header(self, row, before, _data=None):
            if before is None or before.section != row.section:
                row.set_header(label(row.section.upper(), "sidebar-heading",
                                     margin_start=20, margin_top=14 if before else 8,
                                     margin_bottom=4))
            else:
                row.set_header(None)

        def _toast(self):
            self.toast = Gtk.Revealer(halign=Gtk.Align.CENTER, valign=Gtk.Align.END,
                                      margin_bottom=22,
                                      transition_type=Gtk.RevealerTransitionType.SLIDE_UP)
            box = css(Gtk.Box(spacing=10), "toast")
            self.toast_spinner = Gtk.Spinner()
            self.toast_label = label("", max_width_chars=80)
            self.toast_close = Gtk.Button(image=symbolic("window-close-symbolic"))
            self.toast_close.set_relief(Gtk.ReliefStyle.NONE)
            self.toast_close.connect("clicked", lambda *_: self.hide_notice())
            box.pack_start(self.toast_spinner, False, False, 0)
            box.pack_start(self.toast_label, True, True, 0)
            box.pack_start(self.toast_close, False, False, 0)
            self.toast.add(box)
            return self.toast

        def notify(self, text, kind=None, busy=False, autohide=0):
            self._toast_token += 1
            token = self._toast_token
            warn = kind in (Gtk.MessageType.ERROR, Gtk.MessageType.WARNING)
            self.toast_label.set_text(("⚠  " if warn else "") + text)
            self.toast_spinner.set_visible(busy)
            (self.toast_spinner.start if busy else self.toast_spinner.stop)()
            self.toast_close.set_visible(not busy)
            self.toast.set_reveal_child(True)
            if autohide:
                GLib.timeout_add_seconds(autohide, lambda: self._autohide(token))

        def _autohide(self, token):
            if token == self._toast_token:
                self.toast.set_reveal_child(False)
            return False

        def hide_notice(self):
            self._toast_token += 1
            self.toast.set_reveal_child(False)

        def _set_busy(self, busy: bool):
            self._busy = busy
            self.refresh_btn.set_sensitive(not busy)
            self._update_actions()

        def _page_header(self):
            head = Gtk.Box(spacing=16, margin_top=22, margin_start=28, margin_end=28,
                           margin_bottom=6)
            titles = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=4)
            title = label("", "page-title")
            sub = label("", "dim")
            titles.pack_start(title, False, False, 0)
            titles.pack_start(sub, False, False, 0)
            head.pack_start(titles, True, True, 0)
            return head, title, sub

        # ── programs page ────────────────────────────────────────────────────────────────
        def _programs_page(self):
            page = Gtk.Box(orientation=Gtk.Orientation.VERTICAL)
            head, self.page_title, self.page_sub = self._page_header()
            tools = Gtk.Box(spacing=8, valign=Gtk.Align.CENTER)
            self.search = Gtk.SearchEntry(placeholder_text="Search apps and packages…",
                                          width_chars=30)
            self.search.connect("search-changed", lambda *_: self._refilter())
            self.sort = Gtk.ComboBoxText(valign=Gtk.Align.CENTER)
            for key, text in (("name", "Name"), ("size", "Largest first")):
                self.sort.append(key, text)
            self.sort.set_active_id("name")
            self.sort.set_tooltip_text("Sort order")
            self.sort.connect("changed", lambda *_: self._refilter())
            tools.pack_start(self.search, False, False, 0)
            tools.pack_start(self.sort, False, False, 0)
            head.pack_end(tools, False, False, 0)
            page.pack_start(head, False, False, 0)
            self.page_desc = label("", "dim", wrap=True, margin_start=28, margin_end=28,
                                   margin_bottom=16)
            page.pack_start(self.page_desc, False, False, 0)

            content = Gtk.Box(spacing=16, margin_start=28, margin_end=28, margin_bottom=24)
            self.list_stack = Gtk.Stack(transition_type=Gtk.StackTransitionType.CROSSFADE)
            loading = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=12,
                              valign=Gtk.Align.CENTER, halign=Gtk.Align.CENTER)
            spinner = Gtk.Spinner(width_request=32, height_request=32)
            spinner.start()
            loading.pack_start(spinner, False, False, 0)
            loading.pack_start(label("Scanning this computer…", "dim", xalign=0.5),
                               False, False, 0)
            loading_card = css(Gtk.Box(), "card")
            loading_card.pack_start(loading, True, True, 0)
            self.list_stack.add_named(loading_card, "loading")

            # Paged list: filtering/sorting happen in Python over the item dicts and only the
            # first _PAGE matches get widgets (more on scroll) — thousands of GTK rows freeze.
            self.listbox = css(Gtk.ListBox(selection_mode=Gtk.SelectionMode.SINGLE,
                                           margin_top=6, margin_bottom=6), "app-list")
            self.listbox.connect("row-selected", self._on_row_selected)
            self.listbox.set_placeholder(self._empty_state(
                "edit-find-symbolic", "No matches",
                "Nothing here matches your search. Try another word, or look in "
                "“All packages”."))
            self.list_scroll = scrolled(self.listbox, "card")
            self.list_scroll.connect("edge-reached", self._on_edge)
            self.list_stack.add_named(self.list_scroll, "list")
            content.pack_start(self.list_stack, True, True, 0)
            content.pack_start(self._details_pane(), False, False, 0)
            page.pack_start(content, True, True, 0)
            return page

        def _empty_state(self, icon, title, text):
            box = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=8, margin=40,
                          valign=Gtk.Align.CENTER, halign=Gtk.Align.CENTER)
            box.pack_start(css(symbolic(icon, 64), "dim"), False, False, 0)
            box.pack_start(label(title, "empty-title", xalign=0.5), False, False, 0)
            box.pack_start(label(text, "dim", xalign=0.5, wrap=True,
                                 justify=Gtk.Justification.CENTER, max_width_chars=40),
                           False, False, 0)
            box.show_all()
            return box

        def _details_pane(self):
            self.detail_stack = css(Gtk.Stack(width_request=350), "card")
            self.detail_stack.add_named(self._empty_state(
                "system-software-install-symbolic", "No app selected",
                "Pick something from the list to see where it came from and what you can do "
                "with it."), "empty")

            d = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=10, margin=22)
            self.d_icon = Gtk.Image(pixel_size=88, margin_top=6)
            d.pack_start(self.d_icon, False, False, 0)
            self.d_name = label("", "detail-title", xalign=0.5, wrap=True,
                                justify=Gtk.Justification.CENTER)
            d.pack_start(self.d_name, False, False, 0)
            self.d_summary = label("", "dim", xalign=0.5, wrap=True,
                                   justify=Gtk.Justification.CENTER)
            d.pack_start(self.d_summary, False, False, 0)
            self.d_chips = Gtk.Box(spacing=6, halign=Gtk.Align.CENTER)
            d.pack_start(self.d_chips, False, False, 0)

            actions = Gtk.Box(spacing=8, homogeneous=True, margin_top=8)
            self.d_launch = pill_button("Open", icon="media-playback-start-symbolic")
            self.d_launch.connect("clicked", lambda *_: self.on_launch())
            self.d_location = pill_button("Show files", icon="folder-open-symbolic")
            self.d_location.connect("clicked", lambda *_: self.on_open_location())
            self.d_uninstall = pill_button("Uninstall", "destructive-action",
                                           icon="user-trash-symbolic")
            self.d_uninstall.set_tooltip_text("Uninstall (Delete)")
            self.d_uninstall.connect("clicked", lambda *_: self.on_uninstall())
            for b in (self.d_launch, self.d_location, self.d_uninstall):
                actions.pack_start(b, True, True, 0)
            d.pack_start(actions, False, False, 0)

            self.d_warn = css(Gtk.Box(spacing=10, margin_top=4), "warning-box")
            self.d_warn.pack_start(symbolic("dialog-warning-symbolic"), False, False, 0)
            self.d_warn.pack_start(label("Protected system component. Removing it could break "
                                         "your desktop, so it can't be uninstalled here.",
                                         wrap=True), True, True, 0)
            d.pack_start(self.d_warn, False, False, 0)
            self.d_note = css(Gtk.Box(spacing=10, margin_top=4), "info-box")
            self.d_note.pack_start(symbolic("dialog-information-symbolic"), False, False, 0)
            self.d_note_label = label("", wrap=True)
            self.d_note.pack_start(self.d_note_label, True, True, 0)
            d.pack_start(self.d_note, False, False, 0)

            d.pack_start(label("DETAILS", "sidebar-heading", margin_top=10), False, False, 0)
            self.d_props = css(Gtk.Box(orientation=Gtk.Orientation.VERTICAL), "props")
            d.pack_start(self.d_props, False, False, 0)
            self.detail_stack.add_named(scrolled(d), "details")
            return self.detail_stack

        def _show_details(self, item):
            self._current = item
            if not item:
                self.detail_stack.set_visible_child_name("empty")
                return
            self.d_icon.set_from_gicon(gicon_for(item.get("icon", ""), fallback_icon(item)),
                                       Gtk.IconSize.DIALOG)
            self.d_icon.set_pixel_size(88)
            self.d_name.set_text(item["name"])
            self.d_summary.set_text(item.get("summary") or "")
            self.d_summary.set_visible(bool(item.get("summary")))

            for child in self.d_chips.get_children():
                child.destroy()
            self.d_chips.pack_start(badge(item["source"]), False, False, 0)
            if item["source"] == "apt":
                self.d_chips.pack_start(chip("Installed by you" if item.get("manual")
                                             else "Preinstalled"), False, False, 0)
            if item["protected"]:
                self.d_chips.pack_start(chip("Protected", "chip-protected"), False, False, 0)
            self.d_chips.show_all()

            local = item["source"] == "local"
            by = ""
            if item["source"] == "apt":
                by = "You" if item.get("manual") else "Came with the system"
            size = item["size"] if item["size"] not in ("", "-") else ("" if local else "unknown")
            props = [("Type", item.get("kind", "")),
                     ("Package", "" if local else item["id"]),
                     ("Version", "" if local else (item.get("version") or "—")),
                     ("Installed by", by),
                     ("Size on disk", size),
                     ("Installed for", item.get("installation", "")),
                     ("Includes", ", ".join(item.get("includes", []))),
                     ("Location", item.get("location", ""))]
            for child in self.d_props.get_children():
                child.destroy()
            first = True
            for key, value in props:
                if not value:
                    continue
                if not first:
                    self.d_props.pack_start(Gtk.Separator(margin_start=14, margin_end=14),
                                            False, False, 0)
                first = False
                row = Gtk.Box(spacing=12, margin_top=9, margin_bottom=9,
                              margin_start=14, margin_end=14)
                row.pack_start(label(key, "prop-key", ellipsize=False), False, False, 0)
                v = label(value, xalign=1.0, selectable=True)
                v.set_ellipsize(Pango.EllipsizeMode.MIDDLE)
                v.set_tooltip_text(value)
                row.pack_start(v, True, True, 0)
                self.d_props.pack_start(row, False, False, 0)
            self.d_props.show_all()

            self.d_warn.set_visible(item["protected"])
            self.d_note.set_visible(local)
            self.d_note_label.set_text("Not managed by a package manager, so it can't be "
                                       f"uninstalled from here. {item.get('remove_hint', '')}")
            self.d_launch.set_visible(bool(item.get("desktop")))
            self.d_location.set_visible(local and bool(item.get("location")))
            self.d_uninstall.set_visible(not local)
            self._update_actions()
            self.detail_stack.set_visible_child_name("details")

        def _update_actions(self):
            item = self._current
            self.d_uninstall.set_sensitive(bool(item) and not self._busy
                                           and not item["protected"]
                                           and item["source"] != "local")

        # ── filtering / paging ───────────────────────────────────────────────────────────
        def _in_view(self, item, view=None):
            view = view or self._view
            if view == "all":
                return True
            if not item.get("is_app"):
                return False
            return view == "apps" or item["source"] == view

        def _item_visible(self, item):
            if not self._in_view(item):
                return False
            q = self.search.get_text().strip().lower()
            return all(word in item["_hay"] for word in q.split())

        def _sort_key(self, item):
            if self.sort.get_active_id() == "size":
                return (-item.get("size_kb", 0), item["name"].lower())
            return (item["name"].lower(),)

        def _row_for(self, item):
            key = (item["source"], item["id"])
            row = self._row_cache.get(key)
            if row is None:
                row = self._row_cache[key] = ProgramRow(item)
                row.show_all()
            return row

        def _refilter(self):
            self._matches = sorted((i for i in self._rows if self._item_visible(i)),
                                   key=self._sort_key)
            self._rendering = True
            for child in self.listbox.get_children():
                self.listbox.remove(child)  # widgets stay cached in _row_cache
            self._shown = 0
            self._render_more()
            self._rendering = False
            cur = self._current
            if cur is not None and not any(i is cur for i in self._matches):
                self._show_details(None)  # don't keep showing a program the filter hides
            elif cur is not None:
                row = self._row_cache.get((cur["source"], cur["id"]))
                if row is not None and row.get_parent() is self.listbox:
                    self.listbox.select_row(row)
            self.list_scroll.get_vadjustment().set_value(0)
            self._update_page_header()

        def _render_more(self):
            for item in self._matches[self._shown:self._shown + self._PAGE]:
                self.listbox.add(self._row_for(item))
            self._shown = min(len(self._matches), self._shown + self._PAGE)

        def _on_edge(self, _sw, pos):
            if pos == Gtk.PositionType.BOTTOM and self._shown < len(self._matches):
                self._render_more()

        def _update_page_header(self):
            _key, title, _icon, _section, desc = self._views[self._view]
            in_view = [i for i in self._rows if self._in_view(i)]
            noun = "package" if self._view == "all" else "app"
            n = len(in_view)
            kb = sum(i.get("size_kb", 0) for i in in_view)
            sub = f"{n:,} {noun}{'s' if n != 1 else ''}"
            if kb:
                sub += f"  ·  {_human_kb(kb)} on disk"
            q = self.search.get_text().strip()
            if q:
                sub += f"  ·  {len(self._matches):,} matching “{q}”"
            self.page_title.set_text(title)
            self.page_sub.set_text(sub)
            self.page_desc.set_text(desc)

        def _update_counts(self):
            for key, row in self.nav_rows.items():
                if key not in ("startup", "monitor"):
                    row.count.set_text(f"{sum(1 for i in self._rows if self._in_view(i, key)):,}")

        # ── navigation ───────────────────────────────────────────────────────────────────
        def _on_nav(self, _lb, row):
            if row is None:
                return
            if row.key in ("startup", "monitor"):
                self.stack.set_visible_child_name(row.key)
                if row.key == "monitor":
                    self._monitor_start()
                return
            self.stack.set_visible_child_name("programs")
            if row.key != self._view:
                self._view = row.key
                self._refilter()

        def _go(self, key):
            self.nav.select_row(self.nav_rows[key])

        def refresh_current(self):
            page = self.stack.get_visible_child_name()
            if page == "startup":
                self.refresh_startup()
            elif page == "monitor":
                self._monitor_kick()
            else:
                self.refresh_programs()

        def refresh_programs(self):
            self.list_stack.set_visible_child_name("loading")
            keep = (self._current or {}).get("source"), (self._current or {}).get("id")

            def work():
                rows = list_all()
                GLib.idle_add(self._fill_programs, rows, keep)
            threading.Thread(target=work, daemon=True).start()

        def _fill_programs(self, rows, keep=(None, None)):
            for child in self.listbox.get_children():
                self.listbox.remove(child)
            for row in self._row_cache.values():
                row.destroy()
            self._row_cache = {}
            for r in rows:
                r["_hay"] = " ".join((r["name"], r["id"], r.get("summary", ""), r["source"],
                                      r.get("kind", ""), _SOURCE_LABEL.get(r["source"], ""),
                                      *r.get("includes", []))).lower()
            self._rows = rows
            self._current = next((r for r in rows if (r["source"], r["id"]) == keep), None)
            self._update_counts()
            self._refilter()
            self.list_stack.set_visible_child_name("list")
            self._show_details(self._current)
            self.scan_label.set_text(f"{len(rows):,} items · scanned "
                                     f"{GLib.DateTime.new_now_local().format('%H:%M')}")
            return False

        def _on_row_selected(self, _lb, row):
            if not self._rendering:
                self._show_details(row.item if row is not None else None)

        def on_launch(self):
            item = self._current
            if not item or not item.get("desktop"):
                return
            info = Gio.DesktopAppInfo.new_from_filename(item["desktop"])
            try:
                if info is None or not info.launch([], None):
                    raise GLib.Error("could not start")
                self.notify(f"Opening {item['name']}…", autohide=3)
            except GLib.Error as exc:
                self.notify(f"Couldn't open {item['name']}: {exc.message}",
                            Gtk.MessageType.ERROR, autohide=8)

        def on_open_location(self):
            item = self._current
            path = (item or {}).get("location", "")
            folder = path if os.path.isdir(path) else os.path.dirname(path)
            if not folder or not os.path.isdir(folder):
                self.notify("That location no longer exists.", Gtk.MessageType.WARNING,
                            autohide=6)
                return
            try:
                Gio.AppInfo.launch_default_for_uri(Gio.File.new_for_path(folder).get_uri(), None)
            except GLib.Error as exc:
                self.notify(f"Couldn't open {folder}: {exc.message}", Gtk.MessageType.ERROR,
                            autohide=8)

        # ── uninstall flow ───────────────────────────────────────────────────────────────
        def on_uninstall(self):
            item = self._current
            if not item or self._busy or item["source"] == "local":
                return  # not package-managed: nothing we can safely uninstall
            if item["protected"]:
                self._msg(Gtk.MessageType.ERROR, f"“{item['name']}” is protected",
                          "It's an essential system component — removing it could break "
                          "your desktop.")
                return
            self._set_busy(True)
            self.notify(f"Checking what removing {item['name']} would affect…", busy=True)

            def work():
                if item["source"] == "apt":
                    removed, protected = simulate_apt_remove(item["id"])
                    kb = apt_installed_kb(removed)
                else:
                    removed, protected, kb = [item["id"]], [], item.get("size_kb", 0)
                GLib.idle_add(self._confirm_remove, item, removed, protected, kb)
            threading.Thread(target=work, daemon=True).start()

        def _confirm_remove(self, item, removed, protected, kb):
            self._set_busy(False)
            self.hide_notice()
            if item["source"] == "apt" and not removed:
                self._msg(Gtk.MessageType.WARNING, "Nothing to remove",
                          f"apt reports nothing to remove for “{item['id']}”.")
                return False
            if protected:
                self._msg(Gtk.MessageType.ERROR, f"Can't uninstall {item['name']}",
                          "Removing it would also remove essential system package(s):\n\n"
                          + "\n".join(f"  • {p}" for p in protected))
                return False
            ok, purge = self._confirm_dialog(item, removed, kb)
            if ok:
                self._exec_remove(item, purge)
            return False

        def _confirm_dialog(self, item, removed, kb):
            dlg = Gtk.Dialog(transient_for=self, modal=True, title="Uninstall")
            dlg.set_default_size(480, -1)
            dlg.set_resizable(False)
            dlg.add_button("Cancel", Gtk.ResponseType.CANCEL)
            go = css(dlg.add_button("Uninstall", Gtk.ResponseType.OK), "destructive-action")
            dlg.set_default_response(Gtk.ResponseType.CANCEL)
            go.set_can_default(False)

            area = dlg.get_content_area()
            area.set_spacing(14)
            area.set_margin_start(22)
            area.set_margin_end(22)
            area.set_margin_top(18)
            area.set_margin_bottom(10)

            head = Gtk.Box(spacing=14)
            head.pack_start(icon_image(item.get("icon", ""), fallback_icon(item), 56),
                            False, False, 0)
            txt = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=3,
                          valign=Gtk.Align.CENTER)
            txt.pack_start(label(f"Uninstall {item['name']}?", "dialog-title", wrap=True),
                           False, False, 0)
            meta = f"{_SOURCE_LABEL[item['source']]} · {item['id']}"
            if item.get("version"):
                meta += f" · {item['version']}"
            txt.pack_start(label(meta, "dim"), False, False, 0)
            head.pack_start(txt, True, True, 0)
            area.pack_start(head, False, False, 0)

            if kb:
                area.pack_start(label(f"This frees about {_human_kb(kb)} of disk space.",
                                      wrap=True), False, False, 0)

            others = [p for p in removed if p != item["id"]]
            if item["source"] == "apt" and others:
                area.pack_start(label(
                    f"{len(others)} other package{'s' if len(others) != 1 else ''} will also be "
                    "removed because they depend on it or were only installed for it:",
                    wrap=True), False, False, 0)
                lst = Gtk.ListBox(selection_mode=Gtk.SelectionMode.NONE)
                for p in others:
                    lst.add(label(p, margin=4, margin_start=10))
                sw = scrolled(lst, "pkg-list")
                sw.set_min_content_height(min(34 * len(others), 170))
                area.pack_start(sw, False, False, 0)

            purge = None
            if item["source"] == "apt":
                purge = Gtk.CheckButton(label="Also delete its system-wide configuration files")
                area.pack_start(purge, False, False, 0)

            keep = {"apt": "Your personal settings in your home folder are kept.",
                    "flatpak": "Its data in ~/.var/app is kept.",
                    "snap": "snapd saves an automatic snapshot of its data before removal."}
            area.pack_start(label(f"{keep.get(item['source'], '')} You'll be asked for your "
                                  "password to continue.", "dim", wrap=True), False, False, 0)
            dlg.show_all()
            resp = dlg.run()
            do_purge = bool(purge and purge.get_active())
            dlg.destroy()
            return resp == Gtk.ResponseType.OK, do_purge

        def _exec_remove(self, item, purge):
            self._set_busy(True)
            name = item["name"]
            self.notify(f"Uninstalling {name}… waiting for authentication", busy=True)
            argv = remove_command(item, purge)

            def progress(line):
                GLib.idle_add(self.toast_label.set_text, f"Uninstalling {name}… {line[:120]}")

            def work():
                rc, tail = _run_streaming(argv, progress)
                GLib.idle_add(self._done_remove, item, rc, tail)
            threading.Thread(target=work, daemon=True).start()

        def _done_remove(self, item, rc, tail):
            self._set_busy(False)
            if rc == 0:
                self.notify(f"{item['name']} was uninstalled.", autohide=8)
                self._current = None
                self.refresh_programs()
            elif rc in (126, 127):  # pkexec: dialog dismissed / not authorized
                self.notify("Uninstall cancelled — authentication was dismissed or refused.",
                            Gtk.MessageType.WARNING, autohide=8)
            else:
                self.notify(f"Couldn't uninstall {item['name']} (exit code {rc}).",
                            Gtk.MessageType.ERROR, autohide=10)
                self._msg(Gtk.MessageType.ERROR, f"Couldn't uninstall {item['name']}",
                          (tail or "unknown error").strip()[-1500:])
            return False

        # ── startup page ─────────────────────────────────────────────────────────────────
        def _startup_page(self):
            page = Gtk.Box(orientation=Gtk.Orientation.VERTICAL)
            head, title, self.startup_sub = self._page_header()
            title.set_text("Startup apps")
            page.pack_start(head, False, False, 0)
            page.pack_start(label(self._views["startup"][4], "dim", wrap=True, margin_start=28,
                                  margin_end=28, margin_bottom=16), False, False, 0)
            self.au_list = css(Gtk.ListBox(selection_mode=Gtk.SelectionMode.NONE,
                                           margin_top=6, margin_bottom=6), "app-list")
            self.au_list.set_placeholder(self._empty_state(
                "system-run-symbolic", "No startup apps", "Nothing is set to run at login."))
            sw = scrolled(self.au_list, "card")
            sw.set_margin_start(28)
            sw.set_margin_end(28)
            sw.set_margin_bottom(24)
            page.pack_start(sw, True, True, 0)
            return page

        def refresh_startup(self):
            self._au = list_autostart()
            for child in self.au_list.get_children():
                child.destroy()
            for a in self._au:
                self.au_list.add(StartupRow(a, self._on_startup_switch, self._on_startup_delete))
            self.au_list.show_all()
            self._update_startup_header()
            return False

        def _update_startup_header(self):
            on = sum(1 for a in self._au if a["enabled"])
            self.startup_sub.set_text(f"{on} of {len(self._au)} start when you log in")
            self.nav_rows["startup"].count.set_text(str(len(self._au)))

        def _on_startup_switch(self, row, active):
            if self._syncing:
                return
            item = row.item
            try:
                set_autostart_enabled(item, active)
                item["enabled"] = active
                self._update_startup_header()
                self.notify(f"{item['name']} will {'now' if active else 'no longer'} "
                            "start at login.", autohide=4)
            except OSError as exc:
                self._syncing = True
                row.switch.set_active(not active)
                self._syncing = False
                self._msg(Gtk.MessageType.ERROR, "Couldn't change startup setting", str(exc))

        def _on_startup_delete(self, item):
            if self._ask(f"Delete startup entry “{item['name']}”?",
                         "It will no longer start at login. The program itself stays installed.",
                         "Delete"):
                remove_autostart(item)
                self.refresh_startup()
                self.notify(f"Removed “{item['name']}” from startup.", autohide=4)

        # ── system monitor page ──────────────────────────────────────────────────────────
        def _load_viz_css(self):
            mode = "dark" if is_dark(self) else "light"
            self._viz_provider.load_from_data(
                (_METER_CSS % {"accent": _VIZ[mode]["cpu"], **_STATUS}).encode("ascii"))
            return False

        def _mon_card(self, title):
            card = css(Gtk.Box(orientation=Gtk.Orientation.VERTICAL), "card")
            inner = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=10, margin=16)
            head = Gtk.Box(spacing=8)
            head.pack_start(label(title, "app-title", ellipsize=False), False, False, 0)
            value = label("", "dim", "tnum", xalign=1.0)
            head.pack_start(value, True, True, 0)
            inner.pack_start(head, False, False, 0)
            card.pack_start(inner, True, True, 0)
            return card, inner, value

        def _flow(self, children, min_per_line, max_per_line):
            """Cards side by side that wrap onto more rows in a narrow window — the page's
            minimum width is the whole window's minimum, since every page shares one stack."""
            flow = css(Gtk.FlowBox(homogeneous=True, selection_mode=Gtk.SelectionMode.NONE,
                                   min_children_per_line=min_per_line,
                                   max_children_per_line=max_per_line, column_spacing=16,
                                   row_spacing=16, valign=Gtk.Align.START), "mon-flow")
            for child in children:
                flow.add(child)
                child.get_parent().set_can_focus(False)
            return flow

        def _monitor_page(self):
            page = Gtk.Box(orientation=Gtk.Orientation.VERTICAL)
            head, title, self.mon_sub = self._page_header()
            title.set_text("System monitor")
            self.mon_sub.set_text("Taking the first reading…")
            page.pack_start(head, False, False, 0)
            page.pack_start(label(self._views["monitor"][4], "dim", wrap=True, margin_start=28,
                                  margin_end=28, margin_bottom=16), False, False, 0)
            body = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=16, margin_start=28,
                           margin_end=28, margin_bottom=24)

            self.tile_cpu, self.tile_temp = StatTile("CPU usage"), StatTile("CPU temperature")
            self.tile_mem, self.tile_gpu = StatTile("Memory in use"), StatTile("GPU temperature")
            pairs = []  # wrap as pairs: four across, or two by two — never a lone tile
            for a, b in ((self.tile_cpu, self.tile_temp), (self.tile_mem, self.tile_gpu)):
                pair = Gtk.Box(spacing=16, homogeneous=True)
                pair.pack_start(a, True, True, 0)
                pair.pack_start(b, True, True, 0)
                pairs.append(pair)
            body.pack_start(self._flow(pairs, 1, 2), False, False, 0)

            # two small multiples rather than one dual-axis chart: % and °C are different scales
            charts = []
            self.mon_charts = {}
            for key, name, lo, hi, ticks, unit, series in (
                    ("cpu", "CPU usage", 0, 100, (0, 50, 100), "%", "cpu"),
                    ("temp", "CPU temperature", 20, 100, (20, 60, 100), "°", "temp")):
                card, inner, value = self._mon_card(name)
                chart = HistoryChart(self._mon_hist, key, lo, hi, ticks, unit, series,
                                     lambda s, k=key: self._on_chart_hover(k, s))
                inner.pack_start(chart, True, True, 0)
                self.mon_charts[key] = (chart, value)
                charts.append(card)
            body.pack_start(self._flow(charts, 1, 2), False, False, 0)

            left = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=16)
            right = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=16)
            card, inner, self.cpu_info = self._mon_card("Processor")
            self.core_box = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=8)
            inner.pack_start(self.core_box, False, False, 0)
            left.pack_start(card, False, False, 0)
            card, inner, _value = self._mon_card("Memory and storage")
            self.mem_box = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=8)
            inner.pack_start(self.mem_box, False, False, 0)
            left.pack_start(card, False, False, 0)
            card, inner, _value = self._mon_card("Temperatures")
            self.temp_box = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=8)
            inner.pack_start(self.temp_box, False, False, 0)
            self.throttle_label = label("", "dim", wrap=True)
            self.throttle_label.set_no_show_all(True)
            inner.pack_start(self.throttle_label, False, False, 0)
            right.pack_start(card, False, False, 0)
            card, inner, _value = self._mon_card("Graphics")
            self.gpu_box = Gtk.Box(orientation=Gtk.Orientation.VERTICAL, spacing=8)
            inner.pack_start(self.gpu_box, False, False, 0)
            right.pack_start(card, False, False, 0)
            cols = Gtk.Box(spacing=16, homogeneous=True)
            cols.pack_start(left, True, True, 0)
            cols.pack_start(right, True, True, 0)
            body.pack_start(cols, False, False, 0)

            card, inner, _value = self._mon_card("Busiest programs")
            inner.pack_start(label("Share of the whole processor over the last 2 seconds, with "
                                   "each program's processes added together.", "dim", wrap=True),
                             False, False, 0)
            grid = Gtk.Grid(column_spacing=14, row_spacing=8, margin_top=4)
            for col, width, text in ((0, 1, "Program"), (1, 1, "Processes"), (2, 2, "CPU"),
                                     (4, 1, "Memory")):
                grid.attach(label(text, "sidebar-heading", xalign=1.0 if col else 0.0,
                                  ellipsize=False), col, 0, width, 1)
            self.proc_rows = []
            for i in range(8):
                cells = (label("", hexpand=True), label("", "dim", "tnum", xalign=1.0),
                         meter(width_request=110), label("", "tnum", xalign=1.0, width_chars=6),
                         label("", "tnum", xalign=1.0, width_chars=8))
                for col, cell in enumerate(cells):
                    grid.attach(cell, col, i + 1, 1, 1)
                self.proc_rows.append(cells)
            inner.pack_start(grid, False, False, 0)
            body.pack_start(card, False, False, 0)
            page.pack_start(scrolled(body), True, True, 0)
            return page

        def _monitor_start(self):
            self._mon_hist.clear()
            self._mon_throttle0 = None
            self._mon_reset = True  # fresh baseline: rates shouldn't average the time away
            self._monitor_kick()
            GLib.timeout_add(600, self._monitor_kick)  # one-shot: first rates after 0.6 s
            if self._mon_timer is None:
                self._mon_timer = GLib.timeout_add_seconds(_MON_INTERVAL, self._monitor_tick)

        def _monitor_tick(self):
            if self.stack.get_visible_child_name() != "monitor":
                self._mon_timer = None  # stop sampling once the page is left
                self.nav_rows["monitor"].count.set_text("")
                return False
            self._monitor_kick()
            return True

        def _monitor_kick(self):
            if self._mon_busy:
                return False
            self._mon_busy = True
            reset, self._mon_reset = self._mon_reset, False

            def work():
                try:
                    if self._sampler is None:
                        self._sampler = SystemSampler()
                    if reset:
                        self._sampler.reset()
                    snap = self._sampler.sample()
                except (OSError, ValueError, IndexError) as exc:
                    snap = {"error": str(exc)}  # a broken sensor must not take the app down
                GLib.idle_add(self._monitor_apply, snap)
            threading.Thread(target=work, daemon=True).start()
            return False

        def _sync_rows(self, box, names):
            """MeterRows for `names` in `box`, rebuilt only when the set of names changes."""
            rows = self._mon_rows.setdefault(box, {})
            if list(rows) != names:
                for child in box.get_children():
                    child.destroy()
                rows.clear()
                groups = (Gtk.SizeGroup(mode=Gtk.SizeGroupMode.HORIZONTAL),
                          Gtk.SizeGroup(mode=Gtk.SizeGroupMode.HORIZONTAL))
                for name in names:
                    rows[name] = MeterRow(name, groups)
                    box.pack_start(rows[name], False, False, 0)
                box.show_all()
            return rows

        def _on_chart_hover(self, key, sample):
            self._mon_hover[key] = sample
            self._update_chart_value(key)

        def _update_chart_value(self, key):
            _chart, value = self.mon_charts[key]
            unit = "%" if key == "cpu" else " °C"
            sample = self._mon_hover.get(key)
            if sample is not None and sample[key] is not None:
                ago = max(0, round(self._mon_hist[-1]["t"] - sample["t"]))
                value.set_text(f"{sample[key]:.0f}{unit} · {ago} s ago" if ago else
                               f"{sample[key]:.0f}{unit} · now")
            elif self._mon_hist and self._mon_hist[-1][key] is not None:
                value.set_text(f"{self._mon_hist[-1][key]:.0f}{unit} now")
            else:
                value.set_text("No sensor" if key == "temp" and self._mon_hist else "")

        def _monitor_apply(self, snap):
            self._mon_busy = False
            if "error" in snap:
                self.notify(f"Couldn't read the system sensors: {snap['error']}",
                            Gtk.MessageType.WARNING, autohide=8)
                return False
            cpu, mem, ct = snap["cpu"], snap["memory"], snap["cpu_temp"]
            l1, l5, l15 = snap["load"]
            self.mon_sub.set_text(f"Up {_fmt_duration(snap['uptime'])}  ·  load average "
                                  f"{l1:.2f}, {l5:.2f}, {l15:.2f}")
            if cpu["usage"] is not None:
                self._mon_hist.append({"t": snap["time"], "cpu": cpu["usage"],
                                       "temp": ct["c"] if ct else None})
            for key, (chart, _value) in self.mon_charts.items():
                chart.refresh()
                self._update_chart_value(key)

            # headline tiles
            mhz = [c["mhz"] for c in cpu["cores"] if c["mhz"]]
            avg = _fmt_mhz(sum(mhz) / len(mhz) if mhz else None)
            if cpu["usage"] is not None:
                self.tile_cpu.set(f"{cpu['usage']:.0f}%", f"{avg} average · "
                                  f"{len(cpu['cores'])} threads", cpu["usage"] / 100)
            throttle = snap["throttle_count"]
            if ct:
                level, word = temp_level(ct["c"], ct["crit"])
                self.nav_rows["monitor"].count.set_text(f"{ct['c']:.0f}°")
                self.tile_temp.set(f"{ct['c']:.0f} °C", f"{ct['label']} · limit "
                                   f"{ct['crit'] or 100:.0f} °C", ct["c"] / (ct["crit"] or 100),
                                   level, word)
            else:
                self.tile_temp.set("—", "No CPU temperature sensor found", 0)
            used = mem["used_kb"] / mem["total_kb"] if mem["total_kb"] else 0
            level, word = fill_level(used)
            self.tile_mem.set(_human_kb(mem["used_kb"]), f"of {_human_kb(mem['total_kb'])} · "
                              f"{used:.0%}", used, level, word)
            gpus = snap["gpus"]
            gpu = next((g for g in gpus if not g["asleep"]), gpus[0] if gpus else None)
            if gpu is None:
                self.tile_gpu.set("—", "No graphics adapter found", 0, title="Graphics")
            elif gpu["asleep"]:
                self.tile_gpu.set("Asleep", gpu["name"], 0, title="Graphics")
            elif gpu["temp"] is not None:
                level, word = temp_level(gpu["temp"], gpu["temp_crit"])
                shared = gpu["integrated"] and gpu["temp_note"]
                self.tile_gpu.set(f"{gpu['temp']:.0f} °C",
                                  "Same sensor as the CPU" if shared else gpu["name"],
                                  gpu["temp"] / (gpu["temp_crit"] or 100), level, word,
                                  title="GPU temperature")
            else:
                frac = (gpu["mhz"] or 0) / gpu["mhz_max"] if gpu["mhz_max"] else 0
                self.tile_gpu.set(_fmt_mhz(gpu["mhz"]) if gpu["mhz"] else "Idle", gpu["name"],
                                  frac, title="Graphics clock")

            # processor: one row per logical CPU
            self.cpu_info.set_text(f"{cpu['model']} · up to {_fmt_mhz(cpu['mhz_max'])}")
            self.cpu_info.set_tooltip_text(self.cpu_info.get_text())
            names = [f"CPU {int(c['name'][3:]) + 1}" for c in cpu["cores"]]
            rows = self._sync_rows(self.core_box, names)
            for name, core in zip(names, cpu["cores"]):
                if core["usage"] is not None:
                    rows[name].set(core["usage"] / 100,
                                   f"{core['usage']:.0f}% · {_fmt_mhz(core['mhz'])}")

            # memory and storage
            items = [("Memory", used, f"{_human_kb(mem['used_kb'])} / "
                                      f"{_human_kb(mem['total_kb'])}")]
            if mem["swap_total_kb"]:
                swap = mem["swap_used_kb"] / mem["swap_total_kb"]
                items.append(("Swap", swap, f"{_human_kb(mem['swap_used_kb'])} / "
                                            f"{_human_kb(mem['swap_total_kb'])}"))
            for d in snap["disks"]:
                total = d["used_kb"] + d["free_kb"]
                items.append((f"{d['name']} disk", d["used_kb"] / total if total else 0,
                              f"{_human_kb(d['free_kb'])} free"))
            rows = self._sync_rows(self.mem_box, [i[0] for i in items])
            for name, frac, text in items:
                rows[name].set(frac, text, *fill_level(frac))

            # every temperature sensor
            temps = snap["temps"]
            rows = self._sync_rows(self.temp_box, [t["label"] for t in temps])
            for t in temps:
                rows[t["label"]].set(t["c"] / (t["crit"] or 100), f"{t['c']:.0f} °C",
                                     *temp_level(t["c"], t["crit"]))
            if not temps:
                self.throttle_label.set_text("This computer doesn't expose any temperature "
                                             "sensors.")
            elif throttle is not None:
                if self._mon_throttle0 is None:
                    self._mon_throttle0 = throttle
                since = throttle - self._mon_throttle0
                self.throttle_label.set_text(
                    f"The processor has slowed itself down to stay cool {throttle:,} times since "
                    "the computer started" + (f" — {since:,} while this page was open." if since
                                              else "."))
            self.throttle_label.set_visible(not temps or throttle is not None)

            self._apply_gpus(gpus)

            # busiest programs
            procs = snap["processes"]
            for i, cells in enumerate(self.proc_rows):
                p = procs[i] if i < len(procs) else None
                for cell in cells:
                    cell.set_visible(p is not None)
                if p is None:
                    continue
                name, count, bar, cpu_pct, memory = cells
                name.set_text(p["name"])
                name.set_tooltip_text(p["name"])
                count.set_text(str(p["count"]))
                set_meter(bar, p["cpu"] / 100)
                cpu_pct.set_text(f"{p['cpu']:.1f}%")
                memory.set_text(_human_kb(p["memory_kb"]))
            return False

        def _apply_gpus(self, gpus):
            if self._gpu_widgets is None:  # adapters don't change while we run: build once
                self._gpu_widgets = []
                for i, g in enumerate(gpus):
                    if i:
                        self.gpu_box.pack_start(Gtk.Separator(margin_top=4, margin_bottom=4),
                                                False, False, 0)
                    head = Gtk.Box(spacing=8)
                    head.pack_start(label(g["name"], "app-title"), True, True, 0)
                    head.pack_start(chip("Built into the processor" if g["integrated"]
                                         else "Graphics card"), False, False, 0)
                    head.pack_start(chip(f"{g['driver']} driver"), False, False, 0)
                    self.gpu_box.pack_start(head, False, False, 0)
                    rows = {}
                    groups = (Gtk.SizeGroup(mode=Gtk.SizeGroupMode.HORIZONTAL),
                              Gtk.SizeGroup(mode=Gtk.SizeGroupMode.HORIZONTAL))
                    for name in ("Clock speed", "Usage", "Video memory", "Temperature"):
                        rows[name] = MeterRow(name, groups)
                        rows[name].show_all()  # contents visible now; the row itself is
                        rows[name].set_no_show_all(True)  # toggled per reading below
                        rows[name].hide()
                        self.gpu_box.pack_start(rows[name], False, False, 0)
                    note = label("", "dim", wrap=True)
                    note.set_no_show_all(True)
                    self.gpu_box.pack_start(note, False, False, 0)
                    self._gpu_widgets.append((rows, note))
                if not gpus:
                    self.gpu_box.pack_start(label("No graphics adapter found.", "dim"),
                                            False, False, 0)
                self.gpu_box.show_all()
            for g, (rows, note) in zip(gpus, self._gpu_widgets):
                notes = []
                shown = set()
                if g["asleep"]:
                    notes.append("Asleep — powered down to save energy. Readings resume when "
                                 "an app uses it.")
                else:
                    if g["mhz_max"]:
                        mhz = g["mhz"] or 0
                        rows["Clock speed"].set(mhz / g["mhz_max"], f"{mhz:.0f} of "
                                                f"{g['mhz_max']:.0f} MHz" if mhz else "Idle")
                        shown.add("Clock speed")
                    if g["busy"] is not None:
                        rows["Usage"].set(g["busy"] / 100, f"{g['busy']:.0f}%")
                        shown.add("Usage")
                    elif g["driver"] in ("i915", "xe"):
                        notes.append("Intel graphics only report how busy they are to "
                                     "administrators, so the clock speed shows how hard it's "
                                     "working instead.")
                    if g["vram_total_kb"]:
                        frac = (g["vram_used_kb"] or 0) / g["vram_total_kb"]
                        rows["Video memory"].set(frac, f"{_human_kb(g['vram_used_kb'] or 0)} of "
                                                       f"{_human_kb(g['vram_total_kb'])}",
                                                 *fill_level(frac))
                        shown.add("Video memory")
                    if g["temp"] is not None:
                        rows["Temperature"].set(g["temp"] / (g["temp_crit"] or 100),
                                                f"{g['temp']:.0f} °C",
                                                *temp_level(g["temp"], g["temp_crit"]))
                        shown.add("Temperature")
                    if g["temp_note"]:
                        notes.insert(0, g["temp_note"])
                for name, row in rows.items():
                    row.set_visible(name in shown)
                note.set_text(" ".join(notes))
                note.set_visible(bool(notes))

        # ── dialogs & keyboard ───────────────────────────────────────────────────────────
        def _msg(self, kind, title, text=""):
            dlg = Gtk.MessageDialog(transient_for=self, modal=True, message_type=kind,
                                    buttons=Gtk.ButtonsType.OK, text=title)
            if text:
                dlg.format_secondary_text(text)
            dlg.run()
            dlg.destroy()

        def _ask(self, title, text, ok_label):
            dlg = Gtk.MessageDialog(transient_for=self, modal=True,
                                    message_type=Gtk.MessageType.QUESTION,
                                    buttons=Gtk.ButtonsType.NONE, text=title)
            dlg.format_secondary_text(text)
            dlg.add_button("Cancel", Gtk.ResponseType.CANCEL)
            css(dlg.add_button(ok_label, Gtk.ResponseType.OK), "destructive-action")
            dlg.set_default_response(Gtk.ResponseType.CANCEL)
            resp = dlg.run()
            dlg.destroy()
            return resp == Gtk.ResponseType.OK

        def _show_about(self):
            dlg = Gtk.AboutDialog(transient_for=self, modal=True, program_name="App Manager",
                                  logo_icon_name="system-software-install",
                                  comments="See everything installed on this computer — "
                                           "APT, Flatpak, Snap, web apps and more — and "
                                           "uninstall safely.")
            dlg.run()
            dlg.destroy()

        def _show_shortcuts(self):
            self._msg(Gtk.MessageType.INFO, "Keyboard shortcuts",
                      "Type anywhere — search\n"
                      "Ctrl+F — focus search\n"
                      "Delete — uninstall the selected app\n"
                      "F5 or Ctrl+R — rescan (or take a reading now, on System monitor)\n"
                      f"Ctrl+1 … Ctrl+{len(_VIEWS)} — jump to a sidebar section")

        def _on_key(self, _w, event):
            ctrl = event.state & Gdk.ModifierType.CONTROL_MASK
            key = event.keyval
            on_programs = self.stack.get_visible_child_name() == "programs"
            if key == Gdk.KEY_F5 or (ctrl and key in (Gdk.KEY_r, Gdk.KEY_R)):
                if not self._busy:
                    self.refresh_current()
                return True
            if ctrl and key in (Gdk.KEY_f, Gdk.KEY_F):
                if not on_programs:
                    self._go("apps")
                self.search.grab_focus()
                return True
            if ctrl and Gdk.KEY_1 <= key < Gdk.KEY_1 + min(len(_VIEWS), 9):
                self._go(_VIEWS[key - Gdk.KEY_1][0])
                return True
            if isinstance(self.get_focus(), Gtk.Entry) or not on_programs:
                return False
            if key == Gdk.KEY_Delete and self._current:
                self.on_uninstall()
                return True
            # start typing anywhere on the programs page to search
            return self.search.handle_event(event)

    win = AppManager()
    win.show_all()
    win.toast.set_reveal_child(False)
    win._show_details(None)
    win._go("apps")
    Gtk.main()
    return 0


def main(argv: list[str]) -> int:
    if any(a in argv for a in ("--list", "--check", "--json", "--apps", "--monitor")):
        return _cli(argv)
    if "-h" in argv or "--help" in argv:
        print(__doc__)
        return 0
    return run_gui()


if __name__ == "__main__":
    raise SystemExit(main(sys.argv[1:]))
