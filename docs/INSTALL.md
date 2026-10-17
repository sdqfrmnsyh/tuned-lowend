# Install / verify / rollback

How to deploy, check and remove the `tuned-lowend` configuration.

The repository root mirrors `/etc/tuned`, so installation is a file copy plus
activating the profile.

---

## Requirements

- TuneD installed (`tuned` package). Verify: `tuned --version`.
- `sudo` / root.
- The profile is written against an **AMD A4-9125 / `acpi-cpufreq`** target, but
  installs cleanly on any x86_64 system. See [PROFILE-REFERENCE.md](PROFILE-REFERENCE.md)
  for options that are inert depending on hardware.
- Optional: `tuned-ppd` if you use the `ppd.conf` compatibility layer.

---

## Install

### 1. Back up the existing configuration

```sh
sudo cp -a /etc/tuned "/etc/tuned.bak.$(date +%F)"
```

### 2. Copy the files into `/etc/tuned`

From the repository root:

```sh
sudo cp -a \
  tuned-main.conf \
  ppd.conf \
  ppd_base_profile \
  profiles \
  cpu-partitioning-variables.conf \
  cpu-partitioning-powersave-variables.conf \
  realtime-variables.conf \
  realtime-virtual-guest-variables.conf \
  realtime-virtual-host-variables.conf \
  /etc/tuned/
```

> The `tuned-main.conf` and `ppd.conf` files in this repo are the *modified*
> versions. Make sure you are overwriting `/etc/tuned/tuned-main.conf`, not the
> stock copy under `/usr/lib/tuned/` (which does not exist on Arch anyway).

### 2b. Install the systemd unit

[`tuned.service`](../tuned.service) is **not** part of `/etc/tuned`. Install it
as a systemd override so it survives package updates:

```sh
sudo cp tuned.service /etc/systemd/system/tuned.service
sudo systemctl daemon-reload
```

> The reference machine instead has the unit edited in place at
> `/usr/lib/systemd/system/tuned.service`. Both paths work; a unit in
> `/etc/systemd/system/` takes precedence over the packaged one, so prefer it.
>
> This file is what turns TuneD into a one-shot service (`Type=oneshot`,
> `RemainAfterExit=yes`). See the [README](../README.md#the-systemd-unit-tunedservice)
> for the diff against the stock unit.

### 3. Activate the profile

```sh
sudo tuned-adm profile lowend-gaming
sudo systemctl restart tuned
```

In no-daemon mode this command:

1. saves `lowend-gaming` to `/etc/tuned/active_profile` (mode `manual`), and
2. tries to restart the one-shot `tuned.service` to apply it.

> **Arch-specific gotcha:** step 2 is done by calling the `service` helper,
> which **is not installed on Arch**. TuneD therefore prints
> `TuneD (re)start failed, check TuneD logs for details.` The preset from step 1
> is still saved, but nothing is applied until you run:
>
> ```sh
> sudo systemctl restart tuned
> ```
>
> On distributions that provide `service` (RHEL family), the single
> `tuned-adm profile` command is sufficient.

### 4. (Optional) enable the service at boot

The TuneD package normally enables it already:

```sh
sudo systemctl enable --now tuned
```

---

## Verify

```sh
# TuneD's own comparison of the live system against the profile
tuned-adm verify

# Which profile is the preset?
tuned-adm active
tuned-adm profile_mode     # manual

# Service state
systemctl status tuned
systemctl cat tuned | head -3     # confirm which unit file wins

# Was the packaged unit overridden?
ls -l /etc/systemd/system/tuned.service 2>/dev/null
```

With no-daemon mode, `tuned-adm active` prints:

```
It seems that tuned daemon is not running, preset profile is not activated.
Preset profile: lowend-gaming
```

This is the expected message, not an error.

Manual spot checks:

```sh
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor   # performance
cat /sys/devices/system/cpu/cpufreq/policy0/boost           # 1
cat /sys/block/nvme0n1/queue/scheduler                      # [none]
cat /sys/block/nvme0n1/queue/read_ahead_kb                  # 4096
cat /sys/kernel/mm/transparent_hugepage/enabled             # always [madvise] never
sysctl vm.swappiness vm.dirty_ratio kernel.sched_autogroup_enabled
```

> `tuned-adm verify` will report mismatches for the options listed as *ignored*
> in [PROFILE-REFERENCE.md](PROFILE-REFERENCE.md) (`energy_perf_bias`,
> `min_perf_pct`, `max_perf_pct`, `sched_latency_ns`,
> `sched_wakeup_granularity_ns`). Those cannot be applied on the reference
> hardware, so the mismatch is expected.

---

## Switching away from the profile

To go back to a stock profile:

```sh
sudo tuned-adm profile balanced
```

To effectively turn tuning off, switch to the no-op `default` profile:

```sh
sudo tuned-adm profile default
```

> `sudo tuned-adm off` is **not supported in no-daemon mode** — it prints
> `Not supported in no_daemon mode.` Use `tuned-adm profile default` instead, or
> stop the service (`sudo systemctl stop tuned`) and restore the previous
> configuration.

Both `tuned-adm profile` commands save the new preset; on Arch, follow them with
`sudo systemctl restart tuned` (see the [Arch note in step 3](#3-activate-the-profile)).

---

## Rollback to the previous configuration

If you took the backup in step 1:

```sh
sudo systemctl stop tuned

# config
sudo rm -rf /etc/tuned
sudo cp -a /etc/tuned.bak.YYYY-MM-DD /etc/tuned

# systemd unit override (only if you installed it in step 2b)
sudo rm -f /etc/systemd/system/tuned.service
sudo systemctl daemon-reload

sudo systemctl start tuned
```

Then re-select whatever profile was active before.

---

## Uninstall

```sh
sudo systemctl stop tuned
sudo rm -rf /etc/tuned/profiles/lowend-gaming

# restore the previous global config + preset from the backup
sudo cp /etc/tuned.bak.YYYY-MM-DD/tuned-main.conf  /etc/tuned/tuned-main.conf
sudo cp /etc/tuned.bak.YYYY-MM-DD/active_profile   /etc/tuned/active_profile
sudo cp /etc/tuned.bak.YYYY-MM-DD/profile_mode     /etc/tuned/profile_mode

# remove the systemd unit override (restores the packaged unit)
sudo rm -f /etc/systemd/system/tuned.service
sudo systemctl daemon-reload
```

> If you edited the packaged unit in place
> (`/usr/lib/systemd/system/tuned.service`) instead of using an override,
> reinstall the `tuned` package (`sudo pacman -S tuned`) to restore it.

Restore `ppd.conf` / `ppd_base_profile` from the backup as well if you changed
them, then restart the service:

```sh
sudo systemctl restart tuned
```

---

## Editing the profile

- Edit `profiles/lowend-gaming/tuned.conf`.
- Re-apply with `sudo systemctl restart tuned` (recommended on Arch) or
  `sudo tuned-adm profile lowend-gaming` followed by a manual restart.
- Check TuneD's log for errors:

  ```sh
  journalctl -u tuned -n 100 --no-pager
  ```

- Validate syntax with `tuned-adm verify` or by starting TuneD and reading the
  log. TuneD logs a warning line per option it could not apply.

---

## Notes for packaging

If you turn this into a Git repository:

- `active_profile` and `profile_mode` are tracked because they capture the
  desired preset. `post_loaded_profile` and `bootcmdline` are regenerated at
  runtime and are excluded by the included `.gitignore`.
- Consider adding `LICENSE` (the TuneD project is `GPL-2.0-or-later`).
