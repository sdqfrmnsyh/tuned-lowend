# tuned-lowend

**TuneD profiles and global configuration tuned for low-end hardware.**

`tuned-lowend` is a portable, documentation-first snapshot of an `/etc/tuned`
configuration that favours stable, low-latency performance on weak machines
(2 cores, integrated GPU, no modern CPU power-management features) instead of
power saving. The reference target is an **ASUS VivoBook X509BA / M509BA with an
AMD A4-9125 APU**, but the profile is useful for any underpowered x86_64 box
that mostly runs a desktop and light gaming.

> **Name:** the project is called **tuned-lowend** because it is a TuneD
> configuration whose whole reason for existing is low-end specs. The single
> shipped profile is `lowend-gaming`.

---

## Table of contents

- [What this is](#what-this-is)
- [Target hardware](#target-hardware)
- [Design decisions](#design-decisions)
- [Repository layout](#repository-layout)
- [Quick start](#quick-start)
- [The `lowend-gaming` profile](#the-lowend-gaming-profile)
- [Verification](#verification)
- [Known no-ops and caveats](#known-no-ops-and-caveats)
- [Related files outside this repo](#related-files-outside-this-repo)
- [Further reading](#further-reading)
- [License](#license)

---

## What this is

A collection of TuneD (a.k.a. *tuned*) files that:

- force the `performance` CPU governor and disable deep C-state latency,
- switch the NVMe I/O scheduler off (`elevator=none`) and raise read-ahead,
- apply a latency/desktop-oriented set of `vm.*` and `sched_*` knobs,
- run TuneD in **no-daemon (one-shot) mode** so nothing keeps running in the
  background after boot.

It is *not* a fork of TuneD. It only contains configuration: one custom profile
plus a modified global `tuned-main.conf`.

The repository root mirrors `/etc/tuned`, so installing it is a plain copy.
See [docs/INSTALL.md](docs/INSTALL.md) for install/uninstall/rollback steps and
[docs/PROFILE-REFERENCE.md](docs/PROFILE-REFERENCE.md) for an option-by-option
explanation of the profile.

---

## Target hardware

Verified on the reference machine:

| Component | Detail |
| --- | --- |
| Machine | ASUS VivoBook X509BA / M509BA |
| CPU | AMD A4-9125 (Stoney Ridge), 2 cores / 2 threads, 1.3–2.3 GHz |
| CPU scaling driver | `acpi-cpufreq` — **no** EPP, **no** `platform_profile` |
| Boost node | `/sys/devices/system/cpu/cpufreq/policy0/boost` present |
| Root storage | `nvme0n1` (model `KZ256`, 256 GB NVMe) |
| Extra storage | `sda` (WDC WD2500BJKT, 250 GB 2.5" HDD) — *not* tuned |
| OS | Arch Linux |
| Kernel | `7.2.11-1-cachyos` (CachyOS kernel) |
| TuneD | 2.28.0 |

Because the CPU scaling driver is `acpi-cpufreq`, several "modern" TuneD CPU
knobs are silently ignored on this machine. Those are documented explicitly in
[Known no-ops and caveats](#known-no-ops-and-caveats).

---

## Design decisions

These are the choices that make the config low-end friendly. Most live in
[`tuned-main.conf`](tuned-main.conf); the systemd unit that makes the one-shot
model work is covered below.

| Setting | Value | Why |
| --- | --- | --- |
| `daemon` | `0` | No-daemon / one-shot mode. TuneD applies the profile and exits, so no idle daemon eats RAM or wakes the CPU on a 2-core machine. |
| `dynamic_tuning` | `0` | No periodic re-tuning; settings are applied once and left alone. |
| `recommend_command` | `0` | Disables profile recommendation so the manual `lowend-gaming` selection is never silently overridden (no drift back to `balanced`). |
| `reapply_sysctl` | `1` | Makes TuneD re-apply `/etc/sysctl.d/*` **after** its own sysctl block, so files in `/etc/sysctl.d/` win. |
| `profile_dirs` | `/usr/lib/tuned/profiles,/etc/tuned/profiles` | Allows the user profile in `/etc/tuned/profiles`. |
| `startup_udev_settle_wait` | `0` | Skips the udev settle wait for a faster boot. |

The systemd unit runs TuneD as a one-shot service
(`Type=oneshot`, `RemainAfterExit=yes`, `ExecStart=/usr/bin/tuned`). It starts
at boot, applies the active profile, and exits — the service stays "active
(exited)".

### The systemd unit (`tuned.service`)

[`tuned.service`](tuned.service) is the Arch package unit, edited for no-daemon
mode:

| Directive | Stock package | This repo |
| --- | --- | --- |
| `Type` | `dbus` | `oneshot` |
| `RemainAfterExit` | *(absent)* | `yes` |
| `ExecStart` | `/usr/bin/tuned -l -P` | `/usr/bin/tuned` |

- `Type=oneshot` matches a process that applies the profile and exits.
- `RemainAfterExit=yes` keeps the unit in the `active (exited)` state after
  TuneD finishes, so `systemctl status tuned` reports success and units that
  order themselves after `tuned.service` see a completed run.
- Dropping `-l` and `-P` removes file logging (`/var/log/tuned/tuned.log`) and
  the PID file (`/run/tuned/tuned.pid`). With no long-running process there is
  nothing to log for or track; output goes to the journal instead
  (`journalctl -u tuned`).

The `[Unit]` section is unchanged from the package, including
`Conflicts=cpupower.service auto-cpufreq.service tlp.service power-profiles-daemon.service`,
which prevents those competing tuning daemons from running alongside TuneD.

On the reference system this unit is edited in place at
`/usr/lib/systemd/system/tuned.service`. When installing from this repository it
is cleaner to place it at `/etc/systemd/system/tuned.service` (a package
override that survives package upgrades).

---

## Repository layout

The root of this repository is a mirror of `/etc/tuned`.

| Path | Origin | Purpose |
| --- | --- | --- |
| `tuned-main.conf` | **modified** | Global TuneD config (no-daemon mode, sysctl reapply, etc.). |
| `tuned.service` | **modified** (systemd) | One-shot systemd unit — installs to the systemd unit path, **not** into `/etc/tuned/`. |
| `profiles/lowend-gaming/tuned.conf` | **custom** | The `lowend-gaming` profile. |
| `ppd.conf` | **modified** (`tuned-ppd`) | PPD compatibility. Stock `[profiles]`/`[battery]` maps are stripped; `default=lowend-gaming`, `battery_detection=false`. |
| `ppd_base_profile` | **modified** | Base profile for the PPD controller (`balanced`). |
| `active_profile` | **state** (tracked) | Preset profile name (`lowend-gaming`). |
| `profile_mode` | **state** (tracked) | Selection mode (`manual`). |
| `post_loaded_profile` | state (ignored) | Empty; set after a profile is loaded. |
| `bootcmdline` | upstream (ignored) | Kernel cmdline additions; managed by the bootloader plugin. |
| `recommend.d/` | upstream | Recommendation drop-in directory (empty). |
| `cpu-partitioning*.conf` | upstream | Variable templates for the `cpu-partitioning` profiles. |
| `realtime*.conf` | upstream | Variable templates for the `realtime` profiles. |
| `docs/` | new | This documentation. |

The upstream `*variables.conf` files are shipped by the TuneD package and are
kept untouched; they are only relevant if you switch to the
`cpu-partitioning`/`realtime` profiles.

`ppd.conf` belongs to the optional `tuned-ppd` package (a
`power-profiles-daemon` compatibility layer). It is only used when `tuned-ppd`
is installed; the base `tuned` setup ignores it.

> `tuned.service` is the one exception to the "repo root mirrors `/etc/tuned`"
> rule: it is a systemd unit and belongs on the systemd unit path. See
> [The systemd unit](#the-systemd-unit-tunedservice).

---

## Quick start

Full instructions (including uninstall and rollback) live in
[docs/INSTALL.md](docs/INSTALL.md). The short version:

```sh
# 1. Back up the current configuration
sudo cp -a /etc/tuned /etc/tuned.bak.$(date +%F)

# 2. Copy this repository over /etc/tuned
sudo cp -a ./tuned-main.conf ./ppd.conf ./ppd_base_profile ./profiles \
           ./cpu-partitioning*.conf ./realtime*.conf /etc/tuned/

# 2b. Install the systemd unit (it does NOT go into /etc/tuned)
sudo cp ./tuned.service /etc/systemd/system/tuned.service
sudo systemctl daemon-reload

# 3. Set the preset and apply it
sudo tuned-adm profile lowend-gaming
sudo systemctl restart tuned      # see the note below

# 4. Confirm
tuned-adm active
```

With no-daemon mode, `tuned-adm active` is expected to print:

```
It seems that tuned daemon is not running, preset profile is not activated.
Preset profile: lowend-gaming
```

> **Arch note:** `tuned-adm profile` saves the preset first, then restarts the
> service via the `service` helper — which **does not exist on Arch**. The
> preset is still written, but TuneD prints `TuneD (re)start failed` and the
> profile is not applied until you restart the unit yourself:
> `sudo systemctl restart tuned`. On distributions that ship `service` (RHEL
> family), the single `tuned-adm profile` command is enough.

---

## The `lowend-gaming` profile

[`profiles/lowend-gaming/tuned.conf`](profiles/lowend-gaming/tuned.conf) is the
single shipped profile. Summary of its sections:

| Section | Goal |
| --- | --- |
| `[cpu]` | `performance` governor, latency bias, C-state latency limit, boost on. |
| `[disk]` | NVMe I/O scheduler off + larger read-ahead. |
| `[vm]` | THP `madvise`, low swappiness, tuned dirty ratios, larger inotify limits. |
| `[sysctl]` | Kernel-level desktop latency/throughput knobs. |
| `[scheduler]` | EEVDF/CFS scheduler tunables for smoother frame pacing. |

Every option, the exact sysfs/sysctl path it touches, and whether it actually
takes effect on the target hardware is documented in
**[docs/PROFILE-REFERENCE.md](docs/PROFILE-REFERENCE.md)**.

The profile was written for *stable FPS and responsive daily use*, not for
minimum power draw. If your goal is battery life, use a different profile.

---

## Verification

TuneD can compare the live system against the profile:

```sh
tuned-adm verify
```

Manual spot checks:

```sh
# CPU governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor   # performance

# Boost
cat /sys/devices/system/cpu/cpufreq/policy0/boost           # 1

# NVMe scheduler + read-ahead
cat /sys/block/nvme0n1/queue/scheduler                      # [none]
cat /sys/block/nvme0n1/queue/read_ahead_kb                  # 4096

# Transparent huge pages
cat /sys/kernel/mm/transparent_hugepage/enabled             # always [madvise] never

# A few sysctls
sysctl vm.swappiness vm.dirty_ratio kernel.sched_autogroup_enabled
```

`tuned-adm verify` reports failures for the inert knobs described below; that
is expected and not a sign of a broken install.

---

## Known no-ops and caveats

This configuration is deliberately written portably, but some options do **not**
apply on the reference AMD `acpi-cpufreq` machine. They are harmless but worth
knowing about:

| Option | Status | Reason |
| --- | --- | --- |
| `energy_perf_bias=performance` | **Ignored** | TuneD only applies EPB on Intel (via `x86_energy_perf_policy`) or on CPUs with `hwp_epp`. This AMD has neither. |
| `min_perf_pct=100` / `max_perf_pct=100` | **Ignored** | These write to `/sys/devices/system/cpu/intel_pstate/*`, which only exists with the Intel P-state driver. |
| `sched_wakeup_granularity_ns` / `sched_latency_ns` | **Ignored** | Removed from the kernel in 6.6+. On this kernel they no longer exist. |
| `sched_min_granularity_ns` | **Remapped** | Kernel ≥ 6.6 renames it to `base_slice_ns`; TuneD writes to `/sys/kernel/debug/sched/base_slice_ns`. Requires debugfs access (TuneD runs as root, so fine). |
| `sched_migration_cost_ns` | **Relocated** | Written to `/sys/kernel/debug/sched/migration_cost_ns` instead of `/proc/sys/kernel/`. |

The effective CPU levers on this hardware are therefore the **governor**,
**boost** and **`force_latency`**; the P-state percentage knobs are portability
leftovers. See [docs/PROFILE-REFERENCE.md](docs/PROFILE-REFERENCE.md) for the
full per-option table.

Also note that the `[disk]` section scopes itself to `nvme0n1` only. The `sda`
hard disk is intentionally left alone.

---

## Related files outside this repo

The live system has a few tuning-related files that are **not** part of this
repository. If you want a complete snapshot, capture them yourself:

- `/etc/sysctl.d/` — currently **empty**. `tuned-main.conf` is written on the
  assumption that `/etc/sysctl.d/99-lowend.conf` exists and takes precedence
  over TuneD's sysctl block (`reapply_sysctl=1`). If you rely on that file, add
  it and version it separately.
- `/etc/modprobe.d/tuned.conf` — present but empty.
- Kernel command line — the boot line carries `mitigations=off`,
  `usbcore.autosuspend=-1` and `rootflags=data=writeback,commit=60,journal_async_commit`,
  which are part of the same low-end tuning strategy but are managed outside
  TuneD.

> The systemd unit *is* included in this repository as [`tuned.service`](tuned.service);
> see [The systemd unit](#the-systemd-unit-tunedservice).

---

## Further reading

- [docs/PROFILE-REFERENCE.md](docs/PROFILE-REFERENCE.md) — every profile option explained.
- [docs/INSTALL.md](docs/INSTALL.md) — install, verify, rollback, uninstall.
- `man tuned.conf`, `man tuned-main.conf`, `man tuned-adm` — TuneD reference.

---

## License

The configuration in this repository is intended to be distributed under the
same terms as TuneD itself: **GPL-2.0-or-later**. Add a `LICENSE` file if you
publish it as a standalone project.
