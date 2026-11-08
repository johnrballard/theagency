# ADR 0005: Working configuration for a MacBookPro16,1 reinstall

- Date: 2026-11-07
- Status: Accepted; verified in the current desktop session
- Machine: MacBookPro16,1, Intel UHD 630 + AMD Navi 14, Apple T2
- Software: linux-t2 7.2.11.arch1-1, Hyprland 0.56.2, Mesa 26.2.5

## Context

Record the configuration needed to reproduce the working installation:
the internal keyboard works at disk unlock, Intel starts the desktop,
AMD provides accelerated rendering after login, and the speakers use DSP
tuning. This record contains the working settings and reinstall procedure.

## Decision

### 1. Use the T2 kernel and load the keyboard drivers early

Install `linux-t2` and `linux-t2-headers`. Use this kernel for the Limine
UKI. Retain the installer's LUKS, root, Btrfs subvolume, and resume parameters;
their UUIDs and offsets belong to the new installation.

Create `/etc/mkinitcpio.conf.d/zz-omarchy-t2-keyboard.conf`:

```bash
# Keep this after zz-omarchy-kernel-modules.conf in mkinitcpio's load order.
if modinfo -k "${KERNELVERSION:-$(uname -r)}" t2bce_vhci >/dev/null 2>&1; then
  _t2_keyboard_modules=(t2bce_dma t2bce_core t2bce_vhci)
  for _t2_keyboard_module in "${MODULES[@]}"; do
    case "$_t2_keyboard_module" in
      t2bce_dma|t2bce_core|t2bce_vhci) ;;
      *) _t2_keyboard_modules+=("$_t2_keyboard_module") ;;
    esac
  done
  MODULES=("${_t2_keyboard_modules[@]}")
  unset _t2_keyboard_modules _t2_keyboard_module
fi
```

The resulting early module list starts with `t2bce_dma t2bce_core
t2bce_vhci`. Preserve the other installed modules and encryption hooks.
Leave GPU modules out of the explicitly early-loaded `MODULES` list.

### 2. Start on Intel; load AMD manually after reaching the desktop

Keep these graphics parameters in the working UKI command line:

```text
apple_gmux.force_igd=1
modprobe.blacklist=amdgpu
amdgpu.ppfeaturemask=0xfff73fff
amdgpu.dpm=1
```

`modprobe.blacklist=amdgpu` prevents automatic loading while allowing a later
explicit `sudo modprobe amdgpu`. The working profile must not contain
`module_blacklist=amdgpu`, which also prevents manual loading.

On a fresh installation, persist the policy in
`/etc/limine-entry-tool.d/z-t2-amdgpu.conf`:

```bash
KERNEL_CMDLINE[default]+=" apple_gmux.force_igd=1 modprobe.blacklist=amdgpu amdgpu.ppfeaturemask=0xfff73fff amdgpu.dpm=1"
```

Retain the T2 platform parameters already supplied by the installation:

```text
intel_iommu=on iommu=pt pm_async=off mem_sleep_default=deep
```

For an existing Limine setup with per-kernel `KERNEL_CMDLINE[...]` overrides,
put the same graphics parameters in the intended working entry as well.
The verified entry on this installation is named `linux-t2-amd-manual`.
Keep the separate Intel recovery entry and its image available.

Install `/etc/udev/rules.d/30-t2-hybrid-gpu.rules`:

```udev
# Apply low DPM only to the actual AMD DRM card.
ACTION=="add", SUBSYSTEM=="drm", ENV{DEVTYPE}=="drm_minor", KERNEL=="card[0-9]*", DRIVERS=="amdgpu", TEST=="device/power_dpm_force_performance_level", ATTR{device/power_dpm_force_performance_level}="low"

# Stable GPU paths for the compositor.
SUBSYSTEM=="drm", ENV{DEVTYPE}=="drm_minor", KERNEL=="card[0-9]*", DRIVERS=="i915", KERNELS=="0000:00:02.0", SYMLINK+="dri/t2-intel"
SUBSYSTEM=="drm", ENV{DEVTYPE}=="drm_minor", KERNEL=="card[0-9]*", DRIVERS=="amdgpu", KERNELS=="0000:03:00.0", SYMLINK+="dri/t2-amd"
```

In `~/.config/uwsm/env-hyprland`, select Intel first for this boot policy:

```sh
case " $(cat /proc/cmdline) " in
  *" amdgpu.dpm=1 "*)
    if [ -e /dev/dri/t2-intel ]; then
      AQ_DRM_DEVICES=/dev/dri/t2-intel
      if [ -e /dev/dri/t2-amd ]; then
        AQ_DRM_DEVICES="$AQ_DRM_DEVICES:/dev/dri/t2-amd"
      fi
      export AQ_DRM_DEVICES
    fi
    ;;
esac
```

Keep `mesa`, `mesa-utils`, `intel-media-driver`, `vulkan-intel`,
`vulkan-radeon`, `linux-firmware-intel`, and `linux-firmware-amdgpu` installed.
Leave `supergfxctl`/`supergfxd` uninstalled.

Reload udev and rebuild the T2 UKI from the real Btrfs `@` root:

```bash
sudo udevadm control --reload-rules
sudo limine-mkinitcpio linux-t2
```

Verify the resulting boot entry and embedded UKI command line contain the
four graphics parameters above. Power off completely, disconnect external
displays, and boot the Intel/manual-AMD profile. After disk unlock and login:

```bash
sudo modprobe amdgpu
sudo udevadm settle
cat /sys/bus/pci/devices/0000:03:00.0/power_dpm_force_performance_level
```

The DPM value should be `low`. Run an application on AMD explicitly:

```bash
DRI_PRIME=pci-0000_03_00_0 application
```

Ordinary applications continue to use Intel. AMD is loaded manually once
per boot; no automatic AMD-loading service is part of this configuration.

### 3. Use the CPU-backed hardware-cursor buffer

Add to `~/.config/hypr/looknfeel.lua`:

```lua
hl.config({ cursor = { use_cpu_buffer = 1 } })
```

Apply with `hyprctl reload` and confirm `hyprctl configerrors` is empty.

### 4. Install the model-specific speaker DSP

Keep `apple-t2-audio-config`, PipeWire, and WirePlumber installed. Add
`lsp-plugins-lv2` and the AUR package `bankstown` (verified version 1.1.0-2).
The packaged six-channel Apple T2 ALSA profile is retained.

Use the `16_1` speaker graph and impulse responses from
[`lemmyg/t2-apple-audio-dsp`](https://github.com/lemmyg/t2-apple-audio-dsp),
pinned to commit `1895a70350e9444a78c63be20830b4813c5eb0f6`:

```bash
git clone https://github.com/lemmyg/t2-apple-audio-dsp.git
git -C t2-apple-audio-dsp checkout 1895a70350e9444a78c63be20830b4813c5eb0f6
```

From the directory containing that checkout, install the speaker assets and
make the same local graph adjustments:

```bash
python - <<'PY'
from pathlib import Path
import json, shutil
source = Path("t2-apple-audio-dsp/configs/16_1")
target = Path.home() / ".local/share/t2-speaker-dsp/16_1"
target.mkdir(parents=True, exist_ok=True)
for wave in source.glob("*.wav"):
    shutil.copy2(wave, target / wave.name)
graph = json.loads((source / "graph.json").read_text().replace(
    "/usr/share/t2linux-audio/16_1", str(target)))
graph["playback.props"]["target.object"] = "alsa_output.pci-0000_04_00.3.HiFi__Speaker__sink"
graph["capture.props"]["state.default-volume"] = 0.5
for control in graph["filter.graph"]["capture.volumes"]:
    control["min"] = -35.0
(target / "graph.json").write_text(json.dumps(graph, indent=2) + "\n")
PY
```

The two speaker volume controls retain `max = 0.0` and `scale = "cubic"`.
Setting their minimum to `-35.0` makes the middle of the volume slider useful
without raising maximum gain. Keep the upstream speaker filters unchanged.

Create the WirePlumber configuration, expanding `$HOME` for the current user:

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d
cat > ~/.config/wireplumber/wireplumber.conf.d/51-t2-speaker-dsp.conf <<EOF
node.software-dsp.rules = [
  {
    matches = [ { node.name = "alsa_output.pci-0000_04_00.3.HiFi__Speaker__sink" } ]
    actions = {
      create-filter = {
        filter-path = "$HOME/.local/share/t2-speaker-dsp/16_1/graph.json"
        hide-parent = true
      }
    }
  }
]
wireplumber.profiles = { main = { node.software-dsp = required } }
EOF
pactl set-sink-volume alsa_output.pci-0000_04_00.3.HiFi__Speaker__sink 100%
systemctl --user restart wireplumber pipewire pipewire-pulse
```

Select **MacBook Pro T2 DSP Speakers** as the normal output. Its node name is
`audio_effect.t2-161-speakers`. Leave the underlying speaker output at 100%
and adjust listening volume on the DSP output. No microphone DSP is enabled.

## Verification

The current session passed disk unlock, Intel desktop startup, manual AMD
initialization, accelerated AMD OpenGL, and a running graphics demo. Intel
`i915` owns the real `eDP-1` panel. The user confirmed the speaker improvement;
all six DSP output links were verified during playback.

Recheck after reinstall:

```bash
glxinfo -B                                    # Intel, accelerated
DRI_PRIME=pci-0000_03_00_0 glxinfo -B           # AMD, accelerated; after modprobe
cat /sys/module/amdgpu/parameters/ppfeaturemask # 0xfff73fff
hyprctl getoption cursor:use_cpu_buffer         # 1
hyprctl configerrors                          # empty
pactl get-default-sink                         # audio_effect.t2-161-speakers
```

## Consequences

- AMD remains usable after login, with reduced performance from low DPM.
- AMD automatic runtime power-off is not established by this configuration.
- Manual AMD loading is required after each boot. Keep the Intel recovery
  entry until repeated boots and normal use establish longer-term stability.

## Related decisions

- [ADR 0002: Intel/AMD hybrid graphics](0002-t2-intel-amd-hybrid-graphics.md)
- [ADR 0003: AMD GFXOFF, low DPM, and cursor settings](0003-t2-lg-ultrafine-hotplug-stability.md)
