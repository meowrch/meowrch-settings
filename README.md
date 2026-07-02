# meowrch-settings
This repository contains a set of system configurations.
Some of them are inspired by CachyOS.

These settings are designed to improve system performance, speed, and resource management.

# More about optimizations

## Udev Rules: Device Event Automation
Udev rules automatically apply system configurations upon device detection or state changes.
* **Audio Power Management**: Manages `snd-hda-intel` power saving to mitigate audio crackling, disabling it when AC-powered and re-enabling on battery.
* **ZRAM Swap Optimization**: Configures ZRAM to prefer anonymous page compression (`vm.swappiness=150`) and disables Zswap for efficient RAM-based swap.
* **Device Permissions**: Sets `rtc0` and `hpet` device group to "audio" for proper application access.
* **SATA Performance**: Configures SATA host link power management to `max_performance`.
* **I/O Scheduler Assignment**: Dynamically assigns optimal I/O schedulers: `bfq` for HDDs, `mq-deadline` for SATA SSDs, and `kyber` for NVMe SSDs.
* **HDD Performance Tuning**: Applies `hdparm` settings (`-B 254 -S 0`) to rotational disks.
* **NVIDIA Runtime Power Management**: Enables/disables NVIDIA GPU runtime power management on driver bind/unbind events.
* **CPU DMA Latency Access**: Sets permissions for the `cpu_dma_latency` device.

## Sysctl: Kernel Runtime Configuration
Sysctl parameters modify kernel behavior at runtime for system-wide performance and stability.
* **Memory & I/O Management**: Adjusts `vm.swappiness`, `vfs_cache_pressure`, `dirty_bytes`, `dirty_background_bytes`, and `dirty_writeback_centisecs` for balanced memory usage and efficient disk I/O. Disables `vm.page-cluster`.
* **System Stability & Security**: Disables `kernel.nmi_watchdog`, enables `kernel.unprivileged_userns_clone`, restricts `kernel.kptr_restrict`, and disables `kernel.kexec_load_disabled`.
* **Logging & Network**: Configures `kernel.printk` to hide messages from console, increases `net.core.netdev_max_backlog`, and sets `fs.file-max`.

## Modprobe: Kernel Module Parameters
Modprobe configurations control module loading and behavior for hardware-specific optimizations.
* **AMD GPU Driver Enforcement**: Forces `amdgpu` driver for GCN 1.0+ and 2.x GPUs, blacklisting `radeon`.
* **Watchdog Module Blacklist**: Prevents loading of Intel TCO and AMD SP5100 watchdog timers.
* **NVIDIA Driver Optimizations**: 
  * `NVreg_InitializeSystemMemoryAllocations=0` (disables memory clearing for GPU)
  * `NVreg_DynamicPowerManagement=0x02` (mobile GPU power saving)

## Systemd: Service & System Management
Systemd unit and configuration files for streamlined boot, resource management, and service control.
* **Journal Log Limits**: Sets `journald` size limit to 50MB.
* **Service Timeouts**: Defines `DefaultTimeoutStartSec` (15s) and `DefaultTimeoutStopSec` (10s) for services.
* **File Descriptor Limits**: Increases `DefaultLimitNOFILE` for both system (2048:2097152) and user (1024:1048576) services.
* **Time Synchronization**: Configures `systemd-timesyncd` with Cloudflare and Google NTP servers.
* **ZRAM Generator**: Configures ZRAM with `zstd` or `lz4` compression, `ram` size, and `swap-priority=100`.
* **PCI Latency Service**: Enables a systemd service to apply `pci-latency` script at boot.
* **User Service Resource Delegation**: Delegates CPU, cpuset, IO, memory, and pids to user services.

## Tmpfiles: Temporary File & THP Management
Configurations for temporary file cleanup and Transparent Huge Page (THP) behavior.
* **Coredump Retention**: Clears coredumps older than 3 days.
* **THP Defragmentation**: Sets `transparent_hugepage/defrag` to `defer+madvise` for tcmalloc-using applications.
* **THP Shrinker**: Configures `khugepaged/max_ptes_none` for Kernel 6.12+ to optimize THP memory usage.

## Runtime Optimization Scripts
Boot-time scripts in `/usr/bin/` that apply dynamic optimizations requiring hardware detection or runtime adjustments.

### `meowrch-apply-optimizations`
**Purpose**: Applies runtime optimizations that cannot be configured via static files.  
**Systemd Service**: `meowrch-optimizations.service` (enabled at boot)

**Optimizations Applied**:
* **CPU Governor Management**: Automatically sets `performance` governor for all CPU cores when available, ensuring maximum clock speeds and minimal frequency scaling latency.
* **Intelligent I/O Scheduler Selection**: 
  - **SSDs/NVMe**: Assigns `none` scheduler (bypasses kernel I/O scheduling, direct-to-hardware)
  - **HDDs**: Assigns `mq-deadline` scheduler (optimizes for rotational media seek patterns)
  - Detection via `/sys/block/*/queue/rotational` attribute
* **Transparent Huge Pages (THP) Tuning**:
  - Sets THP to `madvise` mode (enabled only for applications explicitly requesting it)
  - Configures THP defrag to `defer` (reduces memory allocation stalls)
* **Network Packet Steering**: 
  - Enables RPS (Receive Packet Steering) with `ff` mask for all network interfaces
  - Distributes network interrupt processing across all available CPU cores
* **GPU Performance Modes**:
  - **NVIDIA**: Activates persistence mode (`nvidia-smi -pm 1`) and locks maximum GPU/memory clocks
  - **AMD**: Forces `manual` power profile and highest performance level via `/sys/class/drm/card0/device/` sysfs interface

### `pci-latency`
**Purpose**: Optimizes PCI device latency timers to reduce audio crackling and improve peripheral responsiveness.  
**Systemd Service**: `pci-latency.service` (enabled at boot)

**Configuration Applied**:
* **Default PCI Devices**: Sets latency timer to `20` cycles (balances throughput and latency)
* **Root Complex (0:0)**: Resets latency timer to `0` (immediate data transfer)
* **Audio Controllers (PCI class 04xx)**: Sets latency timer to `80` cycles (prevents audio buffer underruns and glitches)

Uses `setpci` to directly modify PCI configuration space at boot.

### Systemd Service Integration
**Auto-start Units**:
* `meowrch-optimizations.service` - Executes `/usr/bin/meowrch-apply-optimizations` after `sysinit.target`
* `pci-latency.service` - Executes `/usr/bin/pci-latency` before graphical/multi-user target

Both services run once at boot with `Type=oneshot` and `RemainAfterExit=yes` to maintain optimization state.

## earlyoom: Early Out-Of-Memory Daemon
**Purpose**: Prevents complete system freezes by proactively killing memory-hungry processes before OOM killer activation.  
**Configuration**: `/etc/default/earlyoom-meowrch`

**Protection Strategy**:
* **Memory Threshold**: Triggers at **5% free RAM** (`-m 5`)
* **Swap Threshold**: Triggers at **10% free swap** (`-s 10`)
* **Protected Processes**: System-critical daemons (`init`, `systemd`, `Xorg`, `ssh`, `NetworkManager`, `pipewire`, `wireplumber`)
* **Kill Candidates**: Memory-intensive applications prioritized (`firefox`, `chromium`, `electron`, `code`, `discord`, `telegram`, `steam`)
* **Testing Mode**: Includes `--dryrun` flag by default (remove to enable actual process termination)

Automatically enabled via `earlyoom.service` at boot, ensuring system remains responsive during memory pressure scenarios.

## ananicy-cpp: Automatic Nice Daemon
**Purpose**: Dynamically adjusts process priorities (nice levels, I/O class, OOM scores) based on predefined rules for optimal responsiveness.  
**Rules**: `cachyos-ananicy-rules` - CachyOS-curated ruleset for gaming, desktop, and server workloads

**Key Features**:
* **Game Boost**: Automatically prioritizes game processes with high CPU/IO priority
* **Background Tasks**: De-prioritizes compilers, package managers, and system maintenance
* **Desktop Responsiveness**: Keeps GUI applications responsive under heavy background load
* **OOM Score Adjustment**: Protects important processes from being killed first

## NVIDIA Hyprland VRAM Fix
`/etc/nvidia/nvidia-application-profiles-rc.d/50-limit-free-buffer-pool.json`

Saves 2-4GB VRAM on Wayland by disabling excessive buffer pool
GLVidHeapReuseRatio=0 - NOT fixed in drivers 575+ (Jan 2026)

### Refs
- github.com/hyprwm/Hyprland/issues/7704
- reddit.com/r/hyprland/comments/1m71ukt

# 🙏 Acknowledgements

Optimizations are based on projects:
- [CachyOS](https://cachyos.org/) - for proven performance settings
- [ARU](https://ventureo.codeberg.page/) - Arch Linux optimization guide
- Arch Linux community - for best practices in system configuration