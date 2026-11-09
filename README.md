<div align="center">

![Cuban Eight](cuban-eight/cuban.png)

<br>

[![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![C](https://img.shields.io/badge/C-GCC-A8B9CC?style=for-the-badge&logo=c&logoColor=white)](https://gcc.gnu.org)
[![Platform](https://img.shields.io/badge/Platform-Windows%20WSL-0078D6?style=for-the-badge&logo=windows&logoColor=white)](https://docs.microsoft.com/en-us/windows/wsl/)
[![Version](https://img.shields.io/badge/Version-3.0-FF0000?style=for-the-badge)](https://github.com/init1Security/Cuban-Eight)

<br>

*WSL Distro Packager for Red Team Research*

**Author:** [init1Security](https://github.com/init1Security)

</div>

## Overview

Cuban Eight takes a Windows PE payload and packages it as a `.wsl` file  the double-click installer format used by real WSL distributions. The target opens it, Windows registers it as a Linux distro, and the payload executes immediately through WSL's interop layer.

> **Requirement:** Target must have WSL installed. WSL is not enabled by default  it requires a one-time setup or prior installation.

---

## Demo

<div align="center">

![Demo](cuban-eight/beacon.png)

*Beacon callback captured in Paladin C2 within seconds of the target double-clicking the `.wsl` file*

</div>

<details>
<summary><b>Known behavior on first launch</b></summary>
<br>

```
wsl: Failed to mount C:\, see dmesg for more details.
wsl: Failed to translate 'C:\Users\USER\Desktop'
Press any key to exit...
```

Mount warnings appear briefly because `wsl.conf` only takes effect after OOBE completes. As of v3.0, the console window auto-dismisses after ~1.5 seconds (see **Silent Execution** below).

</details>

---

## How It Works

The `.wsl` format is a gzip-compressed POSIX ustar tar of a Linux rootfs. Cuban Eight builds a scratch rootfs (~35KB uncompressed, ~3KB compressed) with no real Linux userland.

```
Double-click .wsl
       │
       ▼
Windows registers distro (POSIX ustar + gzip + /bin/sh stub passes validation)
       │
       ▼
WSL logs in as root
       │
       ▼
/etc/passwd shell → /root/init.exe (4KB Windows PE wrapper)
       │
       ▼
binfmt_misc intercepts MZ → Windows CreateProcess
       │
       ▼
init.exe: FreeConsole() + CreateProcess(payload.exe, DETACHED_PROCESS)
       │                + CreateProcess(closer.exe, CREATE_NO_WINDOW)
       │
       ▼
init.exe exits → closer.exe sends Enter after 1.5s → console auto-closes
       │
       ▼
Payload running silently on host — no visible window
```

- `/bin/sh` is a 129-byte ELF stub (exits immediately, exists only to pass WSL rootfs validation)
- `init.exe` is a 4KB Windows PE that launches the payload detached and spawns the closer
- `closer.exe` is a 3.5KB Windows PE that sleeps 1.5s then sends a keystroke to dismiss the OOBE prompt
- No shell scripts, no interpreter, no Linux userland
- Compressed package size: ~3KB + payload size

---

## Project Structure

```
cuban-eight/
├── cuban-eight.py      Python frontend (interactive menu)
├── build.sh            Compiles C backend (requires zlib)
└── c/
    ├── cuban8_pack.c   Builds the gzip tar and writes the .wsl
    ├── wrapper_pe.h    Embedded init.exe (4KB Windows PE wrapper)
    ├── closer_pe.h     Embedded closer.exe (3.5KB keystroke sender)
    └── cuban8-pack     Compiled binary (after build)
```

---

## Build & Usage

**Build the backend**

```bash
chmod +x build.sh
./build.sh
# Runs: gcc -O2 -o c/cuban8-pack c/cuban8_pack.c -lz
```

**Run the tool**

```bash
python3 cuban-eight.py
```

The interactive menu walks through:

1. Distro identity to spoof
2. Path to PE payload
3. Internal filename inside the rootfs
4. Output `.wsl` file path

---

## Distro Identities

<div align="center">

| ID | Spoofed Name |
|:---|:---|
| `ubuntu2204` | Ubuntu 22.04.3 LTS |
| `ubuntu2004` | Ubuntu 20.04.6 LTS |
| `debian` | Debian GNU/Linux |
| `kali` | Kali Linux Rolling |
| `opensuse` | openSUSE Leap 15.5 |
| `fedora` | Fedora Linux 39 |
| `alpine` | Alpine Linux |

</div>

---

## Silent Execution

Previous versions left the WSL console window open after payload execution. v3.0 solves this with a three-stage PE chain embedded in the rootfs:

| File | Size | Role |
|:---|:---|:---|
| `init.exe` | 4 KB | Root's login shell. Calls `FreeConsole()`, spawns payload with `DETACHED_PROCESS \| CREATE_NO_WINDOW`, spawns closer, exits. |
| `closer.exe` | 3.5 KB | Sleeps 1.5s, sends `VK_RETURN` via `keybd_event` to dismiss the WSL OOBE "Press any key to exit" prompt. |
| `payload.exe` | varies | The actual implant, running as an independent Windows process with no console. |

**Why a Windows PE wrapper instead of a Linux launcher?**

During WSL's first-launch OOBE, `/mnt/c` is not yet mounted — Linux-side `execve` calls to Windows binaries under `/mnt/c/Windows/System32/` fail with `ENOENT`. However, PE files stored in the WSL rootfs execute via `binfmt_misc` without needing `/mnt/c`, because the kernel's interop handler calls `CreateProcess` directly. This is why `init.exe` lives in `/root/` alongside the payload.

**Why not fork/background from Linux?**

WSL terminates orphaned Linux processes when the login shell exits. A forked child that `execve`s the payload gets killed before execution completes. The Windows-side `DETACHED_PROCESS` flag is the only reliable way to outlive the WSL session.

---

## Detection Notes

- Rootfs contains only the payload and config files  no shell, no standard binaries
- Distro name written to `HKCU\Software\Microsoft\Windows\CurrentVersion\Lxss`
- Payload spawns under `wslhost.exe` or `wsl.exe`
- EDR coverage of WSL-launched processes varies by vendor

**Cleanup:** `wsl --unregister <distro-name>`

---

## Name Inspiration

> *"Pull up. Invert. Roll out. Repeat. Return."*

The **Cuban Eight** is a classic aerobatic maneuver invented by **Len Povey** in 1936  a figure-eight traced horizontally in the vertical plane.

- Pull into a 5/8 loop
- Descend inverted at 45°
- Half-roll to upright
- Pull through another 5/8 loop
- Descend inverted at 45°
- Half-roll to upright
- 1/8 pull-out to level flight

The tool is named after it  one clean execution path, no wasted motion, exits clean.

---

## References

- [WSL BOF Research  MayerDaniel](https://github.com/MayerDaniel/the-one-wsl-bof)
- [Build a Custom WSL Distro  Microsoft Docs](https://learn.microsoft.com/lb-lu/windows/wsl/build-custom-distro)
- [WSL Payloads  dmcxblue](https://github.com/dmcxblue/WSL-Payloads)
- [Modern Initial Access 2026](https://youtu.be/8derKdV8Tbs?si=lORiLr2FKWzWML39)

---

<div align="center">

*For research purposes only.*

</div>
