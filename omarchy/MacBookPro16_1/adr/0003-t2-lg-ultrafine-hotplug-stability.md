# ADR 0003: Stabilize LG UltraFine hotplug on T2 Navi 14

- Date: 2026-08-27
- Status: Accepted
- Machine: MacBookPro16,1, Intel UHD 630 + AMD Navi 14 (`1002:7340`)
- Display: LG UltraFine 4K over direct Thunderbolt 3
- Software: Omarchy 4.0.0, Hyprland 0.56.2, linux-t2 7.1.8

## Context

With Intel driving the internal panel and AMD retained for external output,
hotplugging the UltraFine could freeze input and display output, surge the
fans, and force a shutdown. The terminal kernel evidence was an AMD IP-state
dump followed by an unresponsive SMU and failure to disable GFXOFF:

```text
amdgpu 0000:03:00.0: Dumping IP State
amdgpu 0000:03:00.0: SMU: No response msg_reg: 2a resp_reg: 0
amdgpu 0000:03:00.0: Failed to disable gfxoff!
```

The next boot repaired an unclean EFI filesystem and journals. There was no
OOM kill, userspace core dump, or thermal-trip event.

## Decision

Retain the Intel-primary hybrid configuration from ADR 0002 and apply both
AMD mitigations below.

Disable only the PowerPlay GFXOFF feature in the Limine boot fragment:

```bash
KERNEL_CMDLINE[default]+=" amdgpu.ppfeaturemask=0xfff73fff"
```

The running default was `0xfff7bfff`; clearing `PP_GFXOFF_MASK` bit `0x8000`
produces `0xfff73fff`. GFXOFF alone changed the failure path but did not make
monitor-attached startup safe.

Force AMD DPM to `low` with `/etc/udev/rules.d/30-amdgpu-pm.rules`:

```udev
# T2 MacBook Pro: avoid Navi 14 DPM transitions that can freeze the system.
SUBSYSTEM=="drm", DRIVERS=="amdgpu", ATTR{device/power_dpm_force_performance_level}="low"
```

Use AMD connector `DP-5` for the UltraFine in
`~/.config/hypr/monitors.lua`, at preferred 3840x2160/60 Hz, scale 1.5,
position `0x0`. Keep Intel `eDP-1` centered below it at `512x1440`, scale 2,
and keep the zero-sized AMD ghost connector `eDP-2` disabled.

Use a CPU-backed Hyprland hardware-cursor buffer in
`~/.config/hypr/looknfeel.lua`:

```lua
hl.config({
  cursor = {
    use_cpu_buffer = 1,
  },
})
```

This fixes the reproducible block-shaped cursor after external-display
reconnection without disabling the hardware cursor plane.

## Verification

- The UltraFine authorizes at Thunderbolt 3 40 Gb/s and appears as `DP-5`.
- Hyprland drives it at 3840x2160/60 Hz with scale 1.5.
- Live AMD DPM remains `low`; the PowerPlay mask is `0xfff73fff`.
- A 10-minute soak passed 20 checks with GPU temperature between 52–58°C.
- Three controlled unplug/replug cycles restored the picture within seconds.
- No SMU timeout, GFXOFF failure, GPU reset, ring timeout, page fault,
  watchdog lockup, or critical thermal event occurred.
- The cursor artifact repeated in cycles 1 and 2, then did not recur in cycle
  3 with `cursor:use_cpu_buffer = 1`.
- `hyprctl configerrors` is empty after persisting the layout and cursor fix.

Repeated `Adding stream ... failed with err 28` messages remain during AMD
connector probing and hotplug, but were non-fatal throughout validation.

## Consequences

- The external display now reconnects reliably in the validated workflow.
- AMD is limited to its lowest DPM clock state, reducing discrete-GPU
  performance in exchange for stability.
- Booting with the monitor attached and suspend/resume with it connected have
  not been validated. Boot disconnected and connect after reaching the desktop
  remains the conservative workflow.
- The WD19TB dock was outside this validation; the accepted path is a direct
  Thunderbolt connection.

## Rollback

1. Remove `/etc/udev/rules.d/30-amdgpu-pm.rules` and reboot, or restore AMD
   DPM to `auto` for the current session.
2. Remove `use_cpu_buffer = 1` from `~/.config/hypr/looknfeel.lua`, or restore
   `~/.config/hypr/looknfeel.lua.bak.20260827-cursor`.
3. If necessary, restore
   `/etc/limine-entry-tool.d/z-t2-amdgpu.conf.bak.20260827-gfxoff`, run
   `sudo limine-mkinitcpio`, and reboot with the monitor disconnected.

## References

- [T2 Linux hybrid graphics guide](https://github.com/t2linux/wiki/blob/master/docs/guides/hybrid-graphics.md)
- [Linux AMD PowerPlay feature masks](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/include/amd_shared.h)
- [Linux AMD GFXOFF control path](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c)
- [Linux AMD GPU power controls](https://www.kernel.org/doc/html/latest/gpu/amdgpu/thermal.html)
- [Hyprland 0.56 cursor variables](https://wiki.hypr.land/0.56.0/Configuring/Basics/Variables/)

## Related decisions

- [ADR 0002: T2 Intel/AMD hybrid graphics configuration](0002-t2-intel-amd-hybrid-graphics.md)
