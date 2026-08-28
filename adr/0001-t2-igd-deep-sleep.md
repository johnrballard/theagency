# ADR 0001: Intel gmux + deep S3 on MacBookPro16,1

- Date: 2026-08-26
- Status: Accepted
- Machine: MacBookPro16,1 (T2), Intel UHD 630 + AMD Navi 14 (Radeon Pro 5300M), apple-gmux
- Install: Omarchy 4.0.0, linux-t2, LUKS+btrfs `@`, Limine UKI

## Context

Stock Omarchy on this box put the internal LCD on the AMD GPU via apple-gmux.
Hyprland then died after graphical start: `SMU: No response` → sdma0 timeout →
GPU reset. The T2 sleep drop-in forced `s2idle`, which on this Navi 14 also
failed (SMU −22). The installer cmdline already asked for `mem_sleep_default=deep`.

The LCD mux picks **one** GPU for the panel. Thunderbolt DisplayPort is wired
to the 5300M regardless of mux state.

## Decision

Drive the **internal panel from Intel**. Sleep with **ACPI S3 (`deep`)**. Keep
amdgpu loaded. Do not treat Thunderbolt displays as in-scope.

1. Mux IGD: `apple_gmux.force_igd=1` in the UKI.
2. Disable the T2 `s2idle` sleep drop-in so `deep` is actually used.
3. Disable dummy AMD `eDP-2` (0×0, no EDID) in Hyprland. Do not pin the
   compositor Intel-only (`AQ_DRM_DEVICES`) — that would black-screen TB DP.
4. Do not blacklist amdgpu, do not `amdgpu.modeset=0`, do not
   `i915.enable_guc=3`, do not `apple_gmux.force_idg` (modinfo typo, ignored).

Plymouth was already removed from mkinitcpio HOOKS and `plymouth.enable=0` is
in the UKI (boot hang). That stays. Diagnostic `SyncIntervalSec=1s` was
reverted to stock journald.

## Install value → committed value

| Knob | Install / package default | Committed |
|---|---|---|
| Internal LCD mux | AMD (`03:00.0`) | Intel (`00:02.0`) via `apple_gmux.force_igd=1` |
| `/etc/limine-entry-tool.d/z-t2-amdgpu.conf` | empty (no gmux in UKI) | `KERNEL_CMDLINE[default]+=" apple_gmux.force_igd=1"` |
| `/etc/limine-entry-tool.d/t2-mac.conf` | `mem_sleep_default=deep` | unchanged |
| `/etc/systemd/sleep.conf.d/10-t2-s2idle.conf` | `SuspendState=freeze`, `MemorySleepMode=s2idle` | renamed `.disabled-20260826-011501` |
| `/sys/power/mem_sleep` | fought by that drop-in | `s2idle [deep]` |
| Hyprland `eDP-2` | enabled dummy 0×0 after mux flip | `hl.monitor({ output = "eDP-2", disabled = true })` |
| amdgpu | loaded | loaded (`renderD129` stays) |
| mkinitcpio `plymouth` hook | present | removed |
| `plymouth.enable` | unset | `0` |
| Journal `SyncIntervalSec` | `5m` | `5m` (1s diagnostic removed) |
| Limine `default_entry` | snapshot (2) | **unchanged** — later decision |

## Evidence at accept

- Internal panel: 3/3 unedited entry-1 `@` desktops with 1s gone, no SMU/GPU reset, vs 5/5 SMU deaths with the panel on AMD.
- Soak ~7h on boot `ae7f20c6`, death signatures still 0.
- One deep S3: `PM: suspend entry (deep)` 16:00 → `SMU is resumed successfully!` 16:15, same Hyprland pid.

## How this stays permanent

These files **are** the state. Reboot does not revert them. A UKI rebuild
(`limine-mkinitcpio` on `@` only, never from a snapshot overlay) must keep
`z-t2-amdgpu.conf` and `no-plymouth.conf`.

Fragile against `omarchy update` / `omarchy refresh hyprland`:

- A package may recreate `10-t2-s2idle.conf`. If it does, disable it again.
  Do not run with `s2idle` on this GPU.
- `omarchy refresh hyprland` will drop the `eDP-2` disable; put it back in
  `~/.config/hypr/monitors.lua`.

Do not run `limine-mkinitcpio` from a Snapper overlay. Do not re-add
`enable_guc=3` or `force_idg`.

## Consequences

- Desktop scanout is UHD 630 at 3072×1920, not the 5300M.
- Apps can still use AMD via `renderD129` / `DRI_PRIME` if they opt in.
- Thunderbolt / USB-C displays (LG UltraFine 4K, Dell WD19TB) still belong
  to AMD. Hotplug of the UltraFine is a 1/1 killer (~17s, SMU → GPU reset →
  PCIe tree down). **Do not plug them** until a later ADR.
- Deep sleep is 1/1 menu suspend, not a week of lid cycles.
- Snapshot remains the Limine default until we change `default_entry`.

## Rejected

- Omarchy PR 5129 as written.
- Intel-only `AQ_DRM_DEVICES`.
- `power/control=auto` on `0000:03:00.0`.
- Shipping `SyncIntervalSec=0` or `1s`.
- Making `@` the Limine default in this ADR.
