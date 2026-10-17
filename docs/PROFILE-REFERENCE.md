# Profile reference — `lowend-gaming`

Option-by-option explanation of
[`profiles/lowend-gaming/tuned.conf`](../profiles/lowend-gaming/tuned.conf).

The **Status on target** column reflects the reference machine (AMD A4-9125,
`acpi-cpufreq`, kernel `7.2.11-cachyos`). On other hardware a knob may apply
where it is marked *ignored* here.

---

## `[main]`

```ini
[main]
summary=AMD A4-9125 low-end: stabil FPS gaming + daily
```

Only the human-readable `summary` is set. There is no `include=`, so this
profile does not inherit from any base profile.

---

## `[cpu]`

```ini
[cpu]
priority=10
governor=performance
energy_perf_bias=performance
force_latency=99
boost=1
min_perf_pct=100
max_perf_pct=100
```

| Option | Value | Written to | Status on target |
| --- | --- | --- | --- |
| `priority` | `10` | — (TuneD instance priority) | Applied; orders this plugin instance. |
| `governor` | `performance` | `/sys/devices/system/cpu/cpu*/cpufreq/scaling_governor` | **Applied.** Pins the CPU at the highest frequency and stops the on-demand ramping that causes stutter. |
| `energy_perf_bias` | `performance` | `cpu*/power/energy_perf_bias` (with `hwp_epp`) or `x86_energy_perf_policy` | **Ignored.** TuneD only sets EPB on Intel or on CPUs that advertise `hwp_epp`; this AMD has neither. Harmless portability leftover. |
| `force_latency` | `99` | `/dev/cpu_dma_latency` (PM QoS) | **Applied.** Caps the kernel's CPU DMA latency to 99 µs, which limits how deep the CPU may sleep. Lower = shallower C-states = snappier. Setting it also disables TuneD's dynamic latency tuning. |
| `boost` | `1` | `/sys/devices/system/cpu/cpufreq/policy*/boost` | **Applied.** Keeps frequency boost enabled. |
| `min_perf_pct` | `100` | `/sys/devices/system/cpu/intel_pstate/min_perf_pct` | **Ignored.** `intel_pstate` does not exist on this AMD. |
| `max_perf_pct` | `100` | `/sys/devices/system/cpu/intel_pstate/max_perf_pct` | **Ignored.** Same reason. |

**Effective levers here:** `governor`, `boost`, `force_latency`.

---

## `[disk]`

```ini
[disk]
readahead=>4096
elevator=none
devices=nvme0n1
```

| Option | Value | Written to | Status on target |
| --- | --- | --- | --- |
| `devices` | `nvme0n1` | — | Scopes the whole section to the NVMe root disk only. The `sda` HDD is not touched. |
| `elevator` | `none` | `/sys/block/nvme0n1/queue/scheduler` | **Applied.** Disables the I/O scheduler. NVMe devices have their own queueing; an OS-side scheduler only adds latency. |
| `readahead` | `>4096` | `/sys/block/nvme0n1/queue/read_ahead_kb` | **Applied, increase-only.** The leading `>` means "raise to 4096 KiB (4 MiB) only if it is currently lower — never decrease it". |

---

## `[vm]`

```ini
[vm]
transparent_hugepages=madvise
```

| Option | Value | Written to | Status on target |
| --- | --- | --- | --- |
| `transparent_hugepages` | `madvise` | `/sys/kernel/mm/transparent_hugepage/enabled` | **Applied.** Huge pages are used only when an application explicitly asks (`madvise`), avoiding background defrag/khugepaged work on a 2-core CPU. On the target the live value is `always [madvise] never`. If `transparent_hugepage=` is already on the kernel command line, TuneD leaves it alone. |

---

## `[sysctl]`

```ini
[sysctl]
vm.swappiness=10
vm.vfs_cache_pressure=50
vm.min_free_kbytes=65536
vm.dirty_ratio=15
vm.dirty_background_ratio=5
vm.dirty_expire_centisecs=3000
vm.dirty_writeback_centisecs=1500
vm.page-cluster=0
vm.stat_interval=10
kernel.sched_autogroup_enabled=1
fs.inotify.max_user_watches=524288
fs.inotify.max_user_instances=256
```

Each key is written to `/proc/sys/<key-with-slashes>`.

| Key | Value | Rationale for low-end |
| --- | --- | --- |
| `vm.swappiness` | `10` | Avoid swapping to slow storage until really necessary. |
| `vm.vfs_cache_pressure` | `50` | Keep inode/dentry caches longer, reducing re-reads. |
| `vm.min_free_kbytes` | `65536` | Reserve 64 MiB so the small RAM isn't exhausted under bursty load. |
| `vm.dirty_ratio` | `15` | Cap dirty page cache before synchronous writeback. |
| `vm.dirty_background_ratio` | `5` | Start background writeback early to smooth I/O stalls. |
| `vm.dirty_expire_centisecs` | `3000` | Dirty data may stay in cache for up to 30 s before writeback is required (kernel default; set explicitly for clarity). |
| `vm.dirty_writeback_centisecs` | `1500` | Run the writeback thread every 15 s instead of the 5 s default — fewer timer wakeups. |
| `vm.page-cluster` | `0` | Swap one page at a time; better for random access pattern. |
| `vm.stat_interval` | `10` | Update `/proc/vmstat` every 10 s to cut timer wakeups. |
| `kernel.sched_autogroup_enabled` | `1` | Explicitly enable per-session task grouping for smoother interactive/game responsiveness (this is also the common default). |
| `fs.inotify.max_user_watches` | `524288` | Prevent inotify exhaustion in editors/IDEs/dev tools. |
| `fs.inotify.max_user_instances` | `256` | Raise the per-user inotify instance limit. |

> **Precedence note:** `tuned-main.conf` sets `reapply_sysctl=1`. TuneD applies
> this block first and then re-applies `/etc/sysctl.d/*`. Any file there (e.g.
> `/etc/sysctl.d/99-lowend.conf`) therefore **overrides** the values above. On
> the reference system `/etc/sysctl.d/` is currently empty.

---

## `[scheduler]`

```ini
[scheduler]
sched_min_granularity_ns=3000000
sched_wakeup_granularity_ns=4000000
sched_migration_cost_ns=5000000
sched_latency_ns=12000000
```

These control the CFS/EEVDF CPU scheduler. Values are in nanoseconds.

| Option | Value | Written to | Status on target (kernel ≥ 6.6) |
| --- | --- | --- | --- |
| `sched_min_granularity_ns` | `3000000` | `/sys/kernel/debug/sched/base_slice_ns` | **Applied under the new name.** The kernel renamed `min_granularity_ns` → `base_slice_ns`; TuneD maps it automatically. |
| `sched_latency_ns` | `12000000` | — | **Ignored.** Removed from the kernel in 6.6+. |
| `sched_wakeup_granularity_ns` | `4000000` | — | **Ignored.** Removed from the kernel in 6.6+. |
| `sched_migration_cost_ns` | `5000000` | `/sys/kernel/debug/sched/migration_cost_ns` | **Applied.** Moved from `/proc/sys/kernel/` to debugfs in 6.6+. |

Notes:

- On kernels **older** than 5.13 these were plain `/proc/sys/kernel/sched_*`
  sysctls; TuneD writes to the correct location for the running kernel.
- Writes to `/sys/kernel/debug/sched/*` need debugfs mounted and root access.
  TuneD runs as root, so this works even though the directory is unreadable to
  normal users (`Permission denied` when you `ls` it yourself).
- Because two of the four lines are inert on modern kernels, expect
  `tuned-adm verify` to report these as mismatches. That is cosmetic.

---

## Summary: what actually changes on the reference machine

| Applies | Does not apply |
| --- | --- |
| `governor=performance` | `energy_perf_bias` |
| `boost=1` | `min_perf_pct` / `max_perf_pct` |
| `force_latency=99` | `sched_latency_ns` |
| `[disk]` (NVMe scheduler + read-ahead) | `sched_wakeup_granularity_ns` |
| `[vm]` (THP `madvise`) | |
| all `[sysctl]` keys except any overridden in `/etc/sysctl.d/` | |
| `sched_min_granularity_ns` (as `base_slice_ns`), `sched_migration_cost_ns` | |

The inert options are kept intentionally: the profile is meant to be copied to
other machines, where `intel_pstate`, `hwp_epp` or an older kernel may make them
effective.
