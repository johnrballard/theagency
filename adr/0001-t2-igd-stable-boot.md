# ADR 0001: Stable `@` boot on MacBookPro16,1 via Intel gmux

- Date: 2026-08-26
- Status: Accepted
- Machine: MacBookPro16,1 (T2), Intel UHD 630 + AMD Navi 14 (Radeon Pro 5300M), apple-gmux
- Install: Omarchy 4.0.0, linux-t2, LUKS+btrfs `@`, Limine UKI

## Context

Stock Omarchy left the internal LCD muxed to the AMD GPU. After `graphical.target`,
Hyprland died: `SMU: No response` → sdma0 timeout → GPU reset. Journals at the
stock 5 minute flush looked like a hang before SDDM; they were the same death
with the tail cut off.

Plymouth in the initramfs also SIGSEGV'd on this box and blocked SDDM.

apple-gmux picks **one** GPU for the built-in panel. Parameter name is
`force_igd`. `force_idg` is a modinfo typo and is ignored. `i915.enable_guc=3`
is not valid on Coffee Lake.

## Decision

Boot the live `@` subvolume to Hyprland with the **internal panel on Intel**.

1. UKI cmdline: `apple_gmux.force_igd=1`
   (`/etc/limine-entry-tool.d/z-t2-amdgpu.conf`).
2. Keep Plymouth out of mkinitcpio HOOKS and keep `plymouth.enable=0`
   (`omarchy_hooks.conf`, `no-plymouth.conf`).
3. Rebuild the UKI only on `@` (`limine-mkinitcpio`). Never from a Snapper
   overlay — that mixed T2-2 UKI with T2-3 modules and dropped to vfat emergency.

Do not blacklist amdgpu. Do not add `enable_guc=3` or `force_idg`. Do not
ship journald `SyncIntervalSec=0` or `1s` — that was a camera, not the fix.

## Install value → committed value

| Knob | Install / package default | Committed |
|---|---|---|
| Internal LCD mux | AMD `0000:03:00.0` | Intel `0000:00:02.0` via `apple_gmux.force_igd=1` |
| `/etc/limine-entry-tool.d/z-t2-amdgpu.conf` | empty | `KERNEL_CMDLINE[default]+=" apple_gmux.force_igd=1"` |
| mkinitcpio `plymouth` hook | present | removed |
| `plymouth.enable` | unset | `0` |
| Journal `SyncIntervalSec` | `5m` | `5m` (1s diagnostic removed) |

## Evidence at accept

3/3 unedited Limine entry-1 `@` desktops with `force_igd` and stock journald,
no SMU / GPU reset / sdma timeout. Same kernel, mesa, and firmware as 5/5
SMU deaths with the panel on AMD.

## How this stays permanent

`z-t2-amdgpu.conf` and `no-plymouth.conf` plus HOOKS without plymouth **are**
the state. Reboot does not revert them. A later `limine-mkinitcpio` on `@`
must leave those files in place.

Do not run `limine-mkinitcpio` from a snapshot overlay.

## Consequences

- Scanout is UHD 630 at 3072×1920, not the 5300M.
- Limine default entry is unchanged (still the snapshot). That is a later
  decision, not this one.

## Rejected

- Omarchy PR 5129 as written.
- `i915.enable_guc=3`.
- `apple_gmux.force_idg`.
- Treating `SyncIntervalSec=1s` as part of the shipped config.
