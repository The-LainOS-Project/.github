> ### **Our source code is now hosted at [forgejo.lain.rocks/lainOS](https://forgejo.lain.rocks/lainOS)**

> **lainOS layer 02** is available at https://forgejo.lain.rocks/lainOS/lainOS-layer-02/releases

> **lainOS layer 01** with systemd is available at https://forgejo.lain.rocks/lainOS/lainOS/releases

# LainOS Layer 02: Protocol 7

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Architecture](https://img.shields.io/badge/Arch-x86__64-blue)](https://archlinux.org)
[![Init](https://img.shields.io/badge/Init-OpenRC-green)](https://github.com/OpenRC/openrc)
[![ISO Size](https://img.shields.io/badge/ISO-2.7GB-blue)](https://lainos.net)
[![Status](https://img.shields.io/badge/Status-Stable-green)](https://lainos.net)

<!--Reddit-->
<a href="https://www.reddit.com/r/LainOSdevelopers/" target="_blank">
  <img align="top" src="https://img.shields.io/badge/Reddit-FF4500?style=for-the-badge&logo=reddit&logoColor=white" alt="Reddit">
</a>
<!--Matrix-->
<a href="https://matrix.to/#/#lainos:catgirl.cloud" target="_blank">
  <img align="top" src="https://img.shields.io/badge/Matrix%20-%20%230047a7?style=for-the-badge&logo=matrix" alt="Matrix">
</a>
<!--Discord-->
<a href="https://discord.gg/JdMQvkHqwH" target="_blank">
  <img align="top" src="https://img.shields.io/badge/Discord%20-%20%234900ff?style=for-the-badge&logo=discord" alt="Discord">
</a>
<!--Web page-->
<a href="https://lainos.net" target="_blank">
  <img align="top" src="https://img.shields.io/badge/Lain%20OS%20web-3d3b93?style=for-the-badge&logo=Devbox" alt="Web">
</a>
<!--Forgejo-->
<a href="https://forgejo.lain.rocks/lainOS" target="_blank">
  <img align="top" src="https://img.shields.io/badge/Forgejo-ff6600?style=for-the-badge&logo=forgejo&logoColor=white" alt="Forgejo">
</a>
<br><br>

**A systemd-free Arch Linux derivative built with OpenRC as PID 1, offering full ABI compatibility for systemd-linked software via the Protocol 7 compatibility architecture.**

---

LainOS is a community-driven Linux project led by Grayson Giles ([@amnesia1337](https://github.com/amnesia1337)) and built by developers from the global *Serial Experiments Lain* community. Originally derived from the 2002 LainOS.org coding experiments, the project has evolved into a genuine init-system replacement instead merely a themed respin.

**Layer 02** is our current focus: a daily-driver distribution that balances usability, privacy, and security. Security hardening is a first-class concern, not an afterthought ~ but this is not a specialized security distribution like Qubes or Whonix. It is built to be *usable*, with hardening that does not get in the way.

If you like what we are doing, consider donating: [lainos.net/#donate](https://lainos.net/#donate)

---

## ✨ What Makes Layer 02 Different

| Feature | Implementation |
|---------|---------------|
| **Init System** | OpenRC as PID 1 ~ no systemd binary present |
| **Compatibility** | Protocol 7 layer provides `libsystemd.so.0` ABI via real systemd-libs |
| **Self-Hosted Stack** | Entire OpenRC ecosystem maintained in our own repository |
| **Live ISO** | Fully bootable live environment with Calamares installer |
| **Filesystem** | BTRFS by default, separate ext4 `/boot` for GRUB |
| **Desktop** | Sway 1.12+ tiling compositor with custom keybindings and themed i3status-rs |
| **Security** | Seccomp, mount namespaces, capability dropping, AppArmor MAC, hardened_malloc, ram-wipe |

---

## 🔒 Security Posture

Layer 02 ships with defense in depth at every layer:

- **Protocol 7 Core fuzz tested** ~ dfuzzer 2.6 full interface PASS, AddressSanitizer PASS, libFuzzer 9M+ combined executions, zero crashes
- **Filesystem isolation** ~ hand-rolled mount namespaces (read-only root, private `/tmp`, hidden `/home`/`/root`, minimal `/dev`)
- **Capability bounding set cleared** ~ all five capability fields zeroed after privilege drop
- **Seccomp whitelisting** ~ `socket()` restricted to `AF_UNIX` only; `personality()`/`unshare()`/`setns()` blocked
- **AppArmor MAC** ~ per-daemon profiles for all external-input components, loaded at boot before daemons start
- **hardened_malloc** ~ GrapheneOS light variant, preloaded for sensitive applications
- **RAM wipe** ~ Kicksecure/Whonix-ported dracut shutdown hook + continuous `init_on_alloc`/`init_on_free`
- **Kernel hardening** ~ Full ASLR, ptrace restriction (`yama.ptrace_scope=1`), kexec disabled, unprivileged user namespaces disabled, core dumps disabled, kernel pointer restriction (`kptr_restrict=2`)

**Verify it yourself:**
```bash
doas lainos-security-status              # read-only status dashboard
doas protocol7-core-security-status      # 36-test adversarial suite
```

---

## 🧩 OpenRC Service Isolation & Containment Stack

Layer 02 ships with an **OpenRC-native service isolation stack** that provides systemd-equivalent containment (`ProtectSystem=`, `PrivateTmp=`, capability bounding, resource limits, syscall filtering) without systemd, without compatibility layers, and without forking OpenRC.

The stack composes four kernel primitives per service, driven by declarative `rc_*` variables in `/etc/conf.d/<service>`:

| Layer | Mechanism | What It Does |
|-------|-----------|--------------|
| **Namespace isolation** | bwrap | Mount/PID/network namespace isolation and filesystem containment |
| **Resource limits** | cgroup-v2 | Memory, CPU, and process-count ceilings (`rc_memory_max`, `rc_cpu_quota`, `rc_pids_max`) |
| **Syscall filtering** | seccomp-bpf | Per-service syscall allowlisting (`rc_seccomp_profile`) |
| **Path enforcement** | Landlock LSM | LSM-level path restriction that survives a namespace escape (`rc_landlock_ro`, `rc_landlock_rw`) |

Both `rc-sandbox` and `lainos-sandbox-wrap` are written in **Rust** and compiled to statically-linked, memory-safe binaries. No shell scripts or interpreted code exist in the isolation chain between OpenRC and the target service.

### Default Behavior

All lainOS-shipped services are **sandboxed by default**. A service runs unsandboxed only if explicitly opted out:

```bash
# /etc/conf.d/<service>
rc_sandbox="NO"   # Opt out of all isolation
```

### Service Coverage

The following core services run inside the isolation stack with per-service profiles:

| Service | Seccomp Profile | PID Namespace | Network | Foreground Required |
|---------|-----------------|---------------|---------|---------------------|
| `dnsmasq` | `lainos-network` | Isolated | Host | Yes (`--keep-in-foreground`) |
| `unbound` | `lainos-network` | Isolated | Host | Yes (`-d`) |
| `dnscrypt-proxy` | `lainos-network` | Isolated | Host | Yes (default) |
| `tor` | `lainos-network` | Isolated | Host | Yes (`--RunAsDaemon 0`) |
| `dhcpcd` | `lainos-privileged` | **Host** | Host | No (`rc_unshare_pid="NO"`) |
| `chrony` | `lainos-privileged` | **Host** | Host | Yes (`-d`) |
| `syslog-ng` | `lainos-base` | Isolated | **Isolated** | Yes (`-F`) |
| `acpid` | `lainos-base` | Isolated | Host | No |
| `iwd` | `lainos-privileged` | Isolated | Host | No |

### Declarative Service Configuration

Service isolation is configured entirely through variables in `/etc/conf.d/<service>`:

```bash
# /etc/conf.d/dnsmasq
rc_private_tmp="YES"                    # Private /tmp
rc_protect_home="YES"                   # Hide /home and /root
rc_protect_system="STRICT"              # Read-only /usr and /boot
rc_capability_bounding_set="CAP_NET_BIND_SERVICE,CAP_NET_RAW"
rc_memory_max="256M"                    # Memory limit
rc_pids_max="20"                        # Process count limit
rc_seccomp_profile="lainos-network"     # Syscall allowlist
rc_network_access="YES"                 # Host network namespace
rc_unshare_pid="YES"                    # Isolated PID namespace (foreground required)
```

### Verification

`openrc-security-status` verifies every layer at runtime:

```bash
doas openrc-security-status
```

Example output:

```
=== dnsmasq ===
  process               running as PID 8606 (comm=dnsmasq)
  namespace: mount      ISOLATED (own mount ns)
  namespace: network    shared with host (rc_network_access=YES, correct)
  namespace: pid        ISOLATED (own PID namespace)
  cgroup limits         ENFORCED (memory.max=67108864 pids.max=20)
  seccomp-bpf           ACTIVE (filter mode, 1 filter(s) loaded, no_new_privs set)
  capabilities          NARROWED (CapEff=0x0000000000002400 CapBnd=0x00000000000024c3)
  AppArmor              CONFINED (/usr/bin/dnsmasq)
```

### Failure Behavior

| Layer | Failure Mode | Rationale |
|-------|--------------|-----------|
| bwrap | **Hard failure** ~ service does not start | Primary containment layer |
| seccomp-bpf | **Hard failure** ~ service does not start | Primary containment layer |
| no_new_privs | **Hard failure** ~ service does not start | Closes setuid-based seccomp bypass |
| cgroup-v2 | **Soft failure** ~ service starts with warning | Hardening layer |
| Landlock | **Soft failure** ~ service starts with warning | Backstop layer |

### Security Properties

- **Services are sandboxed by default** ~ opt-out, not opt-in
- **Four independent layers** ~ failure of one does not compromise the others
- **Landlock backstop** ~ LSM-level path enforcement survives a namespace escape (systemd does not have this)
- **Runtime verification** ~ `openrc-security-status` confirms every layer is active and enforcing
- **Rust implementation** ~ memory-safe, statically-linked, no shell scripts in the critical path
- **AppArmor complement** ~ independent path-based MAC layer on top of namespace isolation

---

## 🏗️ Architecture

```
BIOS/UEFI → GRUB/Syslinux → kernel + initramfs
  → Dracut: dmsquash-live mounts squashfs, execs /sbin/openrc-init
     → OpenRC sysinit: dbus, lainos-notifyd, lainos-machine-id
        → OpenRC boot: rfkill-unblock, cgroup-delegate, lainos-ghost-units, syslog-ng
           → OpenRC default: seatd, lainos-dbus-bridge, greetd, chrony, nftables, acpid, polkit
              → greetd → tuigreet → Sway session
                 → lainos-session-sway → lainos-init → Sway
```

### Protocol 7 Compatibility Layer

Protocol 7 is the architectural foundation enabling systemd-free operation while maintaining compatibility with software expecting systemd interfaces.

> **Protocol 7 is not in a position to own your whole system. systemd, by contrast, is.**

Real `systemd-libs` provide ABI compatibility — the client libraries function fine without systemd running as PID 1. `eudev` is a genuine, functional udev implementation. Custom C daemons handle responsibilities that systemd would otherwise own:

| Component | Role |
|-----------|------|
| `lainos-init` | Session initializer ~ detects Wayland, sets environment, execs compositor |
| `lainos-dbus-bridge` | `org.freedesktop.login1` D-Bus facade ~ fuzz tested, runs as nobody |
| `lainos-notifyd` | `sd_notify` socket sink ~ fuzz tested, runs as nobody |
| `lainos-ghost-units` | Creates `/run/systemd/*` ghost directories |
| `lainos-audio-init` | PipeWire + WirePlumber + pipewire-pulse orchestration |
| `lainos-machine-id` | Generates random `/etc/machine-id` on every boot |
| `cgroup-delegate` | cgroup2 mount + controller delegation |

---

## 🖥️ Desktop Experience

- **Sway** tiling compositor with autotiling
- **i3status-rs** themed status bar
- **wofi** application launcher
- **alacritty** terminal emulator (tmux by default)
- **mako** notification daemon
- **swaylock** screen locker with wallpaper background
- **wlogout** session/power menu
- **Powerlevel10k** zsh prompt
- **CoplandOS-GTK** dark theme with StarLabs cursor
- **PipeWire** audio (orchestrated by `lainos-audio-init`)

**Keybindings:**
| Key | Action |
|-----|--------|
| `Mod4+Return` | Open terminal |
| `Mod4+Space` | Application launcher |
| `Mod4+Shift+q` | Close focused window |
| `Mod4+1-9` | Switch workspace |
| `Mod4+w` | Open LibreWolf |
| `Mod4+f` | Open Thunar |

---

## 🛡️ Privacy & Network Stack

- **WiFi off by default** ~ `iwd` does not start automatically; toggle with `wifi on` / `wifi-autostart`
- **MAC randomization** ~ new MAC every time iwd starts; ethernet via `eth0` toggle
- **DNS mediation** ~ centralized via `dnsmasq` at `127.0.0.1:53`
  - **Plaintext** (default) — DHCP with 1.1.1.1/9.9.9.9 fallbacks
  - **Encrypted** ~ `dnsmasq` → `unbound` → `dnscrypt-proxy` (no single component sees both IP and query)
  - **Private** ~ Tor DNSPort via `private-mode`
- **Tor stream isolation** ~ dedicated circuits `tor1`-`tor4` for per-application isolation
- **Optional Tor time sync** ~ `sdwdate` (opt-in, fingerprint-resistant)
- **Pluggable transports** ~ `snowflake`/`obfs4` toggles for censorship resistance
- **nftables** ~ default-deny firewall
- **IPv6 disabled** by default (prevents VPN leaks)
- **Boot clock randomization** ~ ±180 seconds jitter before networking

---

## 📦 Installation

### Live Boot

```bash
doas dd if=~/lainos-out/lainOS-layer-02-*.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

Boot from USB, login as `liveuser` (no password) at `tuigreet`. Calamares launches automatically.

### System Requirements

| | Minimum | Recommended |
|--|---------|-------------|
| CPU | 64-bit x86_64 | 4+ GB RAM |
| RAM | 2 GB | GPU with Mesa drivers |
| Storage | 4 GB USB/disk | USB 3.0 for live boot |

**Tested on:** QEMU/KVM with Virtio GPU, ThinkPad T480 (Libreboot) — baremetal confirmed, UEFI and BIOS, including LUKS FDE.

### Post-Install Quickstart

```bash
wifi on          # Enable WiFi (off by default)
wscan            # Scan and connect with numbered menu
lainos-dns encrypted   # Switch to encrypted DNS
private-mode on      # One-command sensitive-work mode
```

- Privilege escalation: `doas` (not sudo)
- Power menu: `wlogout`
- Lid close: auto-locks with swaylock and suspends
- Quick-start guide opens automatically on first terminal launch: `lainos-quickstart-help`
- Full guide: `lainos-help`

---

## 🛠️ Building

**Build host:** LainOS Layer 02 or Arch Linux with Protocol 7 repository configured

```bash
doas pacman -S archiso base-devel git
git clone https://forgejo.lain.rocks/lainOS/lainos-iso-layer-02.git
cd lainos-iso-layer-02
doas rm -rf ~/lainos-work ~/lainos-out
mkdir -p ~/lainos-work ~/lainos-out
yes "" | doas mkarchiso -v -w ~/lainos-work -o ~/lainos-out protocol7-profile 2>&1 | tee ~/lainos-build.log
lainos-hash-iso
```

ISO output: `~/lainos-out/lainOS-layer-02-YYYY.MM.DD-x86_64.iso`

---

## 🧰 lainos-utils

One-command utilities for daily operation:

| Command | Purpose |
|---------|---------|
| `wifi {on|off|status}` | WiFi radio control |
| `wscan` | iwd scan + connect menu |
| `eth0 {on|off|status}` | Wired interface + MAC randomization |
| `wg1`-`wg4` / `wg1d`-`wg4d` | WireGuard VPN up/down |
| `tor1`-`tor4` | Isolated Tor circuits (auto-detects Electron apps) |
| `private-mode {on|off|status}` | Sensitive-work sequence |
| `lainos-dns {plaintext|encrypted|private|status}` | DNS mode toggle |
| `lainos-sdwdate {enable|disable|status}` | Tor-based time sync |
| `snowflake` / `obfs4` | Tor pluggable transports |
| `ram-wipe {enable|disable|status}` | Shutdown RAM wipe toggle |
| `lainos-hardened-malloc {enable|disable|status}` | System-wide hardened_malloc |

---

## 🤝 Community & Contributing

LainOS is developed by Grayson Giles and the LainOS community.

- **Forgejo:** https://forgejo.lain.rocks/lainOS/
- **Codeberg:** https://codeberg.org/lainOS
- **GitLab:** https://gitlab.com/lainos
- **GitHub:** https://github.com/The-LainOS-Project
- **Website:** https://lainos.net
- **Security verification:** `lainos-security-suite` repository (see org links above)

### Reporting Issues

Please include:
- ISO version/date
- Hardware/VM configuration
- `rc-status` output
- Relevant logs from `/var/log/rc.log` or `dmesg`

---

## 📜 License

LainOS Layer 02 and the Protocol 7 compatibility layer are released under the **GNU General Public License v3.0**.

Individual components (Sway, OpenRC, Calamares, etc.) retain their respective licenses.

---

## 🙏 Acknowledgments

- **Arch Linux** ~ The foundation everything is built on
- **OpenRC** ~ Reliable, predictable init system
- **GrapheneOS** ~ hardened_malloc
- **Sway/wlroots** ~ Modern Wayland compositor ecosystem
- **Calamares** ~ User-friendly system installer
- **Kicksecure/Whonix** ~ sdwdate, bootclockrandomization, ram-wipe, and security-hardening model

---

*Current package: protocol7-core-5.5.3-27*  
*Status: Stable  
*Last updated: 2026-07-29*

Grayson Giles aka amnesia1337  
PGP fingerprint: `456F268D14C9ECCE1A77355803E8F5B63BAC3998`  
Keyserver: https://keys.openpgp.org
