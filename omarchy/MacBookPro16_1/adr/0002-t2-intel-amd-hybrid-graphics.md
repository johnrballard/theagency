# ADR 0002: T2 Intel/AMD hybrid graphics configuration

- Date: 2026-08-27
- Status: Accepted
- Machine: MacBookPro16,1 (T2), Intel UHD 630 (`8086:3e9b`) + AMD Navi 14 (`1002:7340`)
- Install: Omarchy 4.0.0, Hyprland 0.56.2, linux-t2 7.1.8, Limine UKI

## Context

The MacBookPro16,1 contains an Intel integrated GPU, an AMD discrete GPU,
and an Apple gmux. The desired configuration is:

- Intel owns the internal panel and renders the normal desktop.
- AMD remains initialized and available for applications that explicitly
  request it.
- The AMD driver is not blacklisted.
- Apple gmux, rather than Omarchy's generic laptop GPU toggle, selects the
  primary display adapter.

ADR 0001 established that routing the internal panel through Intel avoids
the AMD display-path failures seen during the original installation. This
record extends that decision to the complete hybrid-graphics setup.

## Decision

### Kernel and gmux

Use the T2-patched kernel and force the integrated GPU at apple-gmux probe:

```text
apple_gmux.force_igd=1
```

For this Limine installation, keep the parameter in
`/etc/limine-entry-tool.d/z-t2-amdgpu.conf`:

```bash
KERNEL_CMDLINE[default]+=" apple_gmux.force_igd=1"
```

Keep the existing T2 platform parameters:

```text
intel_iommu=on iommu=pt pm_async=off mem_sleep_default=deep
```

Do not use the misspelled `apple_gmux.force_idg` parameter. Do not add
`i915.enable_guc=3` unless it is needed to address a demonstrated
suspend/resume problem; it is not part of the baseline configuration.

### Graphics drivers and application selection

Keep both kernel drivers enabled:

- `i915` for Intel UHD 630
- `amdgpu` for AMD Navi 14

Keep the corresponding userspace components installed:

- `mesa`
- `intel-media-driver`
- `vulkan-intel`
- `vulkan-radeon`
- `linux-firmware-intel`
- `linux-firmware-amdgpu`

Hyprland uses the Intel render node by default. Launch an application on
the AMD GPU explicitly with:

```bash
DRI_PRIME=1 application
```

Do not globally set `DRI_PRIME=1`, because that would move ordinary
desktop applications back onto the discrete GPU.

### Omarchy hybrid-GPU control

Do not install, enable, or use `supergfxctl`/`supergfxd` on this Mac.
Omarchy's Hybrid GPU toggle is implemented with `supergfxctl` for
supported PC laptop switching implementations (the source calls out the
ASUS G14); it does not operate Apple gmux.

Remove the unused package and the configuration it created:

```bash
sudo pacman -Rns supergfxctl
sudo rm /etc/supergfxd.conf
```

The removal does not affect `linux-t2`, `apple_gmux`, `i915`, or
`amdgpu`.

### AMD power management

For the validated LG UltraFine external-display configuration, ADR 0003
supersedes the balanced `auto` baseline below and requires DPM `low` plus a
GFXOFF override.

Leave `power_dpm_force_performance_level` at `auto` for the baseline so
the AMD GPU remains available at normal performance through
`DRI_PRIME=1`.

If heat or battery drain becomes more important than performance, the T2
Linux guide documents two optional policies:

1. Set the AMD DPM performance level to `low`. This keeps the GPU
   available but limits its performance.
2. Power the AMD GPU off through `vgaswitcheroo`. This saves substantially
   more power, but the GPU is unavailable for PRIME offload until it is
   re-enabled and the machine is rebooted.

Neither optional policy is part of the balanced baseline.

## Verification

The accepted configuration was observed as follows:

- `/proc/cmdline` contains `apple_gmux.force_igd=1`.
- The kernel reports `apple_gmux: Switching to IGD`.
- Intel `i915` owns connected panel `eDP-1` at 3072x1920.
- Hyprland holds Intel `renderD128` as its rendering device.
- AMD `amdgpu` initializes successfully with 4080 MiB VRAM and exposes
  `renderD129`.
- No process normally holds the AMD render node.
- Both Intel and Radeon Vulkan ICDs are installed.

The AMD driver may expose a zero-sized `eDP-2` connector and log EDID or
display-stream errors after gmux routes the real panel to Intel. In this
configuration those messages do not mean AMD compute/render offload
failed; the meaningful checks are successful `amdgpu` initialization and
the presence of its render node.

Useful checks after a kernel or graphics-stack update:

```bash
cat /proc/cmdline
lspci -nnk | grep -A3 -E 'VGA|3D|Display'
readlink -f /sys/class/drm/renderD*/device/driver
hyprctl monitors all
journalctl -b -k | grep -E 'apple_gmux|i915|amdgpu'
```

## Consequences

- The Intel GPU handles normal desktop rendering and internal-panel
  scanout.
- The AMD GPU remains usable on demand rather than driving the whole
  desktop.
- The discrete GPU can remain powered even while its render node is idle;
  this is the cost of keeping it immediately available.
- Omarchy's generic Hybrid GPU menu is not the authority for GPU mode on
  this machine.

## References

- [T2 Linux hybrid graphics guide](https://github.com/t2linux/wiki/blob/master/docs/guides/hybrid-graphics.md)
- [T2 Linux apple-gmux `force_igd` kernel patch](https://github.com/t2linux/linux-t2-patches/blob/main/2009-apple-gmux-allow-switching-to-igpu-at-probe.patch)
- [Omarchy Hybrid GPU toggle implementation](https://github.com/basecamp/omarchy/blob/quattro/bin/omarchy-toggle-hybrid-gpu)
- [Omarchy Vulkan driver selection](https://github.com/basecamp/omarchy/blob/quattro/install/hardware/vulkan.sh)
- [Omarchy Intel video-acceleration setup](https://github.com/basecamp/omarchy/blob/quattro/install/hardware/intel/video-acceleration.sh)

## Related decisions

- [ADR 0001: Internal LCD on Intel for a clean `@` boot](0001-t2-igd-stable-boot.md)
- [ADR 0003: Stabilize LG UltraFine hotplug on T2 Navi 14](0003-t2-lg-ultrafine-hotplug-stability.md)
