> ### **Our source code is now hosted at [forgejo.lain.rocks/lainOS](https://forgejo.lain.rocks/lainOS)**

> **lainOS layer 03** is available at https://forgejo.lain.rocks/lainOS/lainOS-layer-03/releases

> **lainOS layer 02** with Protocol 7 is available at https://forgejo.lain.rocks/lainOS/lainOS-layer-02/releases

# LainOS Layer 03: Circuit VIII

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Architecture](https://img.shields.io/badge/Arch-x86__64-blue)](https://gentoo.org)
[![Init](https://img.shields.io/badge/Init-OpenRC-green)](https://github.com/OpenRC/openrc)
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

**A systemd-free Gentoo-based Linux distribution built with OpenRC as PID 1, delivering compile-time hardening, split-controller DNS mediation, and native OpenRC service isolation.**

---

LainOS is a community-driven Linux project led by Grayson Giles ([@amnesia1337](https://github.com/amnesia1337)) and built by developers from the global *Serial Experiments Lain* community. Originally derived from the 2002 LainOS.org coding experiments, the project has evolved from an arch respin into a Gentoo-based distribution.

**Layer 03** is our next-generation focus: a daily-driver distribution that balances usability, privacy, and security, migrating the foundation from Arch Linux to Gentoo. Security hardening is a first-class concern, baked into the build itself via compile-time flags and a modular 4-layer OpenRC process isolation stack. This is not a specialized security distribution like Qubes or Whonix. It is built to be *usable*, with hardening that does not get in the way.

If you like what we are doing, consider donating: [lainos.net/#donate](https://lainos.net/#donate)

---

## ✨ What Makes Layer 03 Different

| Feature | Implementation |
|---------|---------------|
| **Init System** | OpenRC as PID 1 ~ no systemd binary present |
| **Service Isolation** | Native OpenRC sandboxing via Rust (`rc-sandbox`, `lainos-sandbox-wrap`), bwrap, cgroup-v2, seccomp, and Landlock |
| **Deployment** | Pre-built SquashFS image deployed to disk, ensuring deterministic, drift-free installations |
| **Live ISO** | Fully bootable live environment with Calamares installer |
| **Filesystem** | BTRFS by default, separate ext4 `/boot` for GRUB |
| **Desktop** | Sway 1.12+ tiling compositor with custom keybindings and themed i3status-rs |
| **Security** | Compile-time hardening (PIE, Full RELRO, FORTIFY_SOURCE=3, CFI, LTO), AppArmor MAC, hardened_malloc, ram-wipe |

---

## 🔒 Security Posture

Layer 03 ships with defense in depth at every layer:

- **Compile-time hardening** ~ PIE, Full RELRO, FORTIFY_SOURCE=3, stack protection, CFI, and LTO set as compiler defaults in the Portage profile
- **OpenRC process isolation stack** ~ written in Rust, composing bwrap (namespaces), cgroup-v2 (resource limits), seccomp-bpf (syscall filtering), and Landlock (path enforcement)
- **Capability bounding set cleared** ~ minimal kernel capabilities, dropped by `lainos-sandbox-wrap` before its syscall filter loads
- **AppArmor MAC** ~ per-daemon profiles for the full stack (DNS, networking, media, crypto, browsers), loaded at boot before daemons start
- **hardened_malloc** ~ GrapheneOS light variant, preloaded for sensitive applications
- **RAM wipe** ~ Kicksecure/Whonix-ported dracut shutdown hook + continuous `init_on_alloc`/`init_on_free`
- **Kernel hardening** ~ Full ASLR, ptrace restriction (`yama.ptrace_scope=1`), kexec disabled, unprivileged user namespaces disabled, core dumps disabled, kernel pointer restriction (`kptr_restrict=2`)

**Verify it yourself:**
```bash
doas lainos-security-status              # read-only status dashboard for kernel hardening, etc.
doas openrc-security-status              # Adversarial test suite for OpenRC process isolation verification
```

---

## 🧩 OpenRC Service Isolation & Containment Stack

Layer 03 ships with a **native OpenRC service isolation stack** that provides systemd-equivalent containment (`ProtectSystem=`, `PrivateTmp=`, capability bounding, resource limits, syscall filtering) without systemd, without compatibility layers, and without forking OpenRC.

The stack composes four kernel primitives per service, driven by declarative `rc_*` variables in `/etc/conf.d/<service>`:

| Layer | Mechanism | What It Does |
|-------|-----------|--------------|
| **Namespace isolation** | bwrap | Mount/PID/network namespace isolation and filesystem containment |
| **Resource limits** | cgroup-v2 | Memory, CPU, and process-count ceilings (`rc_memory_max`, `rc_cpu_quota`, `rc_pids_max`) |
| **Syscall filtering** | seccomp-bpf | Per-service syscall allowlisting (`rc_seccomp_profile`) |
| **Path enforcement** | Landlock LSM | LSM-level path restriction that survives a namespace escape (`rc_landlock_ro`, `rc_landlock_rw`) |

Both `rc-sandbox` and `lainos-sandbox-wrap` are written in **Rust** and compiled to statically-linked, memory-safe binaries. No shell scripts or interpreted code exist in the isolation chain between OpenRC and the target service.

### Default Behavior

All lainOS-shipped core services are **sandboxed by default**. A service runs unsandboxed only if explicitly opted out:

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
| `dhcpcd` | `lainos-privileged` | **Host** (`rc_unshare_pid="NO"`) | Host | No |
| `chrony` | `lainos-privileged` | **Host** (`rc_unshare_pid="NO"`) | Host | Yes (`-d`) |
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

```text
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

## DNS Mediation Architecture

LainOS provides a localized, stateless DNS forwarding architecture built around `dnsmasq`. Rather than exposing upstream resolver information directly to applications, the operating system presents a single, stable resolver endpoint (`127.0.0.1:53`) and centralizes DNS policy within a dedicated forwarding layer.

Applications never communicate directly with upstream DNS servers. The resolver architecture remains identical regardless of operational mode; only the forwarding destination changes.

### Architecture

All DNS resolution follows a single deterministic path:

```text
                +----------------------+
                |   Local Application  |
                +----------+-----------+
                           |
                           v
                    /etc/resolv.conf
                           |
                           v
                      127.0.0.1:53
                           |
                           v
                        dnsmasq
                           |
               +-----------+-----------+-----------+
               |                       |           |
               |                       |           |
         Plaintext Mode          Encrypted Mode  Private Mode
               |                       |           |
               v                       v           v
     DHCP / Manual Fallback     Local Proxy    Tor DNSPort
     (1.1.1.1, 9.9.9.9)        (127.0.0.1:5053)  (9059)
                                  |
                                  v
                               unbound
                           (127.0.0.1:5053)
                                  |
                                  v
                            dnscrypt-proxy
                           (127.0.0.1:5300)
                                  |
                    +-------------+-------------+
                    |                           |
                    v                           v
              Anonymized Relay              Resolver
              (IP hiding)              (DNSCrypt, no-log)
```

### Stateless Forwarding

`dnsmasq` operates as a forwarding resolver rather than a caching resolver:

```text
cache-size=0
no-negcache
```

This intentionally avoids retaining successful or negative DNS query history in memory. All caching, prefetching, and TTL management is delegated to `unbound`. If compromised, `dnsmasq` leaks no historical query data.

The resolver is bound exclusively to the loopback interface and is never exposed externally.

### Operational Modes

| Mode | Description | Activation |
|------|-------------|------------|
| **Plaintext** (Default) | DHCP-provided resolver with fallbacks to 1.1.1.1/9.9.9.9 | `lainos-dns plaintext` |
| **Encrypted** | `dnsmasq` → `unbound` (DNSSEC, caching) → `dnscrypt-proxy` (encrypted transport, anonymized relay) | `lainos-dns encrypted` |
| **Private** | All DNS through Tor's DNSPort (127.0.0.1:9059) | `private-mode on` |

### Split-Controller Privacy in Encrypted Mode

In encrypted mode, the DNS chain splits responsibilities so no single component sees both the user's IP and their plaintext query:

- **`dnsmasq`** ~ sees the user's IP but has no cache and no query history
- **`unbound`** ~ validates DNSSEC, caches, and forwards to dnscrypt-proxy; does not see the user's IP
- **`dnscrypt-proxy`** ~ encrypts and routes through an anonymized relay; sees neither the query nor the user's IP
- **Relay** ~ knows the user's IP but not the query
- **Resolver** ~ knows the query but not the user's IP

This is a stronger privacy model than direct DoT or DoH to a single provider, where one entity sees both.

### Mode Transitions

The `lainos-dns` utility manages transitions between plaintext and encrypted modes, persisting the active mode to `/var/lib/lainos/dns-mode`.

---

## **Operational Modes**

### **Plaintext Mode (Default)**

During normal operation, `dnsmasq` forwards requests to the resolver information provided by DHCP. Manual fallback servers are available for situations where DHCP has not yet completed.

Typical upstream hierarchy:
- DHCP-provided DNS
- Cloudflare (`1.1.1.1`)
- Quad9 (`9.9.9.9`)

Activated by: `lainos-dns plaintext`

---

### **Encrypted Mode**

All DNS queries are routed through the `unbound` → `dnscrypt-proxy` chain with anonymized relay routing. Queries are encrypted, the user's IP is hidden from the resolver, and DNSSEC is validated locally.

Activated by: `lainos-dns encrypted`

---

### **Private Mode**

`private-mode` is a single command that switches the whole system into a hardened, Tor-routed state: WiFi/Ethernet MAC addresses are randomized, Snowflake bridges and `sdwdate` (Tor-based time sync) are enabled, and all DNS resolution is routed exclusively through Tor's own DNSPort — no plaintext or DoT DNS traffic leaves the system while private mode is active.

Usage

```
doas private-mode on
doas private-mode off
doas private-mode status
```

(`private-mode` auto-elevates via `doas` if not already run as root, so the explicit `doas` above is optional.)

What `on` does

1. Locks out `lainos-wifi-dns` (via a lock file it checks for and exits on) so it can't independently re-trigger DNS backends later in this sequence, then stops DNS backends (`unbound`, `dnscrypt-proxy`, `stubby`) directly.
2. Takes WiFi and Ethernet down, waiting for the default route to genuinely disappear before continuing.
3. Enables the Snowflake pluggable transport and `sdwdate`.
4. Brings WiFi and Ethernet back up with randomized MAC addresses. On WiFi, this does not reconnect automatically — open a second terminal and run `wscan` to reconnect once the interface is back up.
5. Waits for a real network route (up to 60s), then always bootstraps Tor's own DNS resolution via encrypted DNS (`dnscrypt-proxy` + `unbound`/`stubby`), regardless of which DNS mode was active before entering private mode. If the encrypted path can't come up, `private-mode on` aborts outright rather than ever falling back to plaintext — no unencrypted DNS traffic leaves the system at any point while private mode is active, including during bootstrap.
6. Once Tor's DNSPort responds, switches `dnsmasq` to route exclusively through Tor's DNSPort, stops `unbound`/`dnscrypt-proxy`, and releases the `lainos-wifi-dns` lock — they serve no purpose once fully on Tor DNS, so nothing unnecessary keeps running during your most privacy-sensitive mode.

The whole sequence typically takes up to ~2 minutes (Tor DNSPort readiness is polled for up to 120s); `private-mode on` exits once Tor is fully bootstrapped. Use `nyx` after reconnecting to verify the Tor connection directly.

What `off` does

Takes the network down (again with a real route-down wait, not just an interface-state check), disables `sdwdate`, stops Tor, and restores whichever DNS mode (plaintext or encrypted) was active before private mode was originally entered — not simply defaulting back to plaintext. Network stays down afterward; run `wifi on` or `eth0 on` to reconnect manually, and DNS backends will start automatically once connectivity returns.

`status`

Shows Snowflake, `sdwdate`, WiFi, and Ethernet status together with the current DNS mode (`plaintext`, `encrypted`, or `private`).

---


### AppArmor Confinement and Service Isolation

The entire DNS forwarding layer is confined under AppArmor along with having its OpenRC services isolated:

- `/usr/bin/dnsmasq` ~ limited to loopback networking and necessary configuration paths
- `/usr/bin/unbound` ~ restricted to resolver operations and cache directories
- `/usr/bin/dnscrypt-proxy` ~ restricted to encrypted socket operations and relay lists

These profiles reduce attack surface by limiting filesystem access, network capabilities, and system calls to only what is required for DNS forwarding operations.

### Commands

```bash
lainos-dns plaintext    # Plaintext fallbacks
lainos-dns encrypted    # Encrypted: unbound + dnscrypt-proxy
lainos-dns private      # Tor DNSPort, via private-mode
lainos-dns status       # Show current mode and full chain status
```

Mode transitions are explicit and stateful; `private-mode` remembers and restores your previous mode on exit.

---

## Summary

The DNS mediation architecture provides:

- **Stateless forwarding** ~ `dnsmasq` has no cache and no query history
- **Loopback isolation** ~ resolver never exposed externally
- **Split-controller privacy** ~ no single component sees both IP and query
- **Explicit mode transitions** ~ user always knows which mode is active
- **AppArmor confinement** ~ all DNS daemons are AppArmor-enforced
- **Application transparency** ~ applications see only `127.0.0.1:53`

---

## 🏗️ Architecture

#### Boot/Session chain

```
Firmware (BIOS/Libreboot or UEFI)
  → GRUB (i386-pc or x86_64-efi target)
    → kernel + dracut initramfs (mounts real root, loads drivers)
      → switch_root → openrc-init (PID 1)
openrc-init
  → sysinit: dbus, lainos-machine-id
  → boot: syslog-ng, cgroup-delegate lainos-apparmor
  → default:
      dhcpcd → dnsmasq → unbound → dnscrypt-proxy → tor   (all rc-sandbox/bwrap-isolated)
      acpid, chrony/sdwdate                               (rc-sandbox/bwrap-isolated)
      iwd
      seatd, nftables, polkit                            (NOT bwrap-isolated — rfkill needs real /dev)
      greetd → tuigreet (login prompt)
tuigreet (after login)
  → lainos-session-sway (C binary)
      sets GSETTINGS_BACKEND=keyfile, XDG_RUNTIME_DIR, XDG_CURRENT_DESKTOP=sway
      lainos-apply-theme (first login only)
      → dbus-run-session -- lainos-init (C binary)
          sets LIBSEAT_BACKEND=seatd, SHELL=<pw_shell>, preserves a whitelisted
          env subset, selects compositor
          → sway (Wayland compositor) → user's bar/terminal/apps
```


---

## 🖥️ Desktop Experience

- **Sway** tiling compositor with autotiling
- **i3status-rs** themed status bar
- **wofi** application launcher
- **alacritty** terminal emulator (tmux by default)
- **mako** notification daemon
- **swaybg** static wallpaper
- **swaylock** screen locker with wallpaper background
- **wlogout** session/power menu
- **Powerlevel10k** zsh prompt
- **CoplandOS-GTK** dark theme with StarLabs cursor
- **PipeWire** audio, bundled by default

**Keybindings:**
| Key | Action |
|-----|--------|
| `Mod4+Return` | Open terminal (alacritty + tmux) |
| `Mod4+Space` | Application launcher (wofi) |
| `Mod4+Shift+q` | Close focused window |
| `Mod4+1-9` | Switch workspace |
| `Mod4+Shift+1-9` | Move window to workspace |
| `Mod4+h/j/k/l` | Focus left/down/up/right |
| `Mod4+Shift+h/j/k/l` | Move window left/down/up/right |
| `Mod4+w` | Open LibreWolf |

---

## 🛡️ Privacy & Network Stack

- **WiFi off by default** ~ `iwd` does not start automatically; toggle with `wifi on` / `wifi-autostart`
- **MAC randomization** ~ new MAC every time iwd starts; ethernet via `eth0` toggle
- **DNS mediation** ~ centralized via `dnsmasq` at `127.0.0.1:53`
  - **Plaintext** (default) ~ DHCP with 1.1.1.1/9.9.9.9 fallbacks
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
doas dd if=~/lainos-out/lainOS-layer-03-*.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

Boot from USB, login as `liveuser` (no password) at `tuigreet`. Calamares launches automatically.

### System Requirements

| | Minimum | Recommended |
|--|---------|-------------|
| CPU | 64-bit x86_64 | 4+ GB RAM |
| RAM | 2 GB | GPU with Mesa drivers |
| Storage | 4 GB USB/disk | USB 3.0 for live boot |

**Tested on:** QEMU/KVM with Virtio GPU, ThinkPad T480 (Libreboot) ~ baremetal confirmed, UEFI and BIOS, including LUKS FDE with BTRFS.

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

**Build host:** Gentoo Linux with OpenRC, or any Linux distribution capable of chroot operations.

```bash
emerge -uv app-portage/ebuild app-portage/gentoolkit sys-fs/squashfs-tools sys-boot/grub sys-kernel/dracut
git clone https://forgejo.lain.rocks/lainOS/lainos-iso-layer-03.git
cd lainos-iso-layer-03
doas ./build-lainos.sh
```

ISO output: `/var/tmp/lainos-out/lainOS-layer-03-YYYY.MM.DD-x86_64.iso`

---

## 🧰 lainos-utils

One-command utilities for daily operation:

| Command | Purpose |
|---------|---------|
| `wifi {on\|off\|status}` | WiFi radio control |
| `wscan` | iwd scan + connect menu |
| `eth0 {on\|off\|status}` | Wired interface + MAC randomization |
| `wg1`-`wg4` / `wg1d`-`wg4d` | WireGuard VPN up/down |
| `tor1`-`tor4` | Isolated Tor circuits (auto-detects Electron apps) |
| `private-mode {on\|off\|status}` | Sensitive-work sequence |
| `lainos-dns {plaintext\|encrypted\|private\|status}` | DNS mode toggle |
| `lainos-sdwdate {enable\|disable\|status}` | Tor-based time sync |
| `snowflake` / `obfs4` | Tor pluggable transports |
| `ram-wipe {enable\|disable\|status}` | Shutdown RAM wipe toggle |
| `lainos-hardened-malloc {enable\|disable\|status}` | System-wide hardened_malloc |
| `maybenot-tunnel {enable\|disable\|status}` | Traffic analysis resistance |
| `kloak` | Keystroke anonymization |

---

## 🤝 Community & Contributing

LainOS is developed by Grayson Giles and the LainOS community.

- **Forgejo:** https://forgejo.lain.rocks/lainOS/
- **Website:** https://lainos.net
- **Security verification:** `lainos-security-suite` repository (see org links above)

### Reporting Issues

Please include:
- ISO version/date
- Hardware/VM configuration
- `rc-status` output
- Relevant logs from `/var/log/daemon.log` or `dmesg`

---

## 📜 License

LainOS Layer 03 is released under the **GNU General Public License v3.0**.

Individual components (Sway, OpenRC, Calamares, etc.) retain their respective licenses.

---

## 🙏 Acknowledgments

- **Gentoo Linux** ~ The foundation everything is built on
- **OpenRC** ~ Reliable, predictable init system
- **GrapheneOS** ~ hardened_malloc
- **Sway/wlroots** ~ Modern Wayland compositor ecosystem
- **Calamares** ~ User-friendly system installer
- **Kicksecure/Whonix** ~ sdwdate, bootclockrandomization, ram-wipe, and security architecture inspiration

---

*Status: Stable*  
*Last updated: 2026-10-09*

Grayson Giles aka amnesia1337  
PGP fingerprint: `456F268D14C9ECCE1A77355803E8F5B63BAC3998`  
Keyserver: https://keys.openpgp.org
