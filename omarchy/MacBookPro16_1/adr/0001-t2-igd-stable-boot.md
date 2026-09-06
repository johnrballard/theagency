# ADR 0001: Internal LCD on Intel for a clean `@` boot

- Date: 2026-08-26
- Status: Accepted
- Machine: MacBookPro16,1 (T2), Intel UHD 630 + AMD Navi 14 (Radeon Pro 5300M), apple-gmux
- Install: Omarchy 4.0.0, linux-t2, LUKS+btrfs `@`, Limine UKI

## Context

Omarchy's default T2 install leaves apple-gmux on the discrete GPU. The
built-in panel then comes up as AMD `0000:03:00.0`. After
`graphical.target`, Hyprland dies: `SMU: No response` → sdma0 timeout →
GPU reset. That is the boot failure this record covers.

apple-gmux selects **one** GPU for the internal LCD. The module parameter
is `force_igd`. `i915.enable_guc=3` is not valid on Coffee Lake.

## Decision

Mux the **internal panel to Intel** at boot so i915 owns scanout.

Add one kernel cmdline token, baked into the UKI:

```
apple_gmux.force_igd=1
```

in `/etc/limine-entry-tool.d/z-t2-amdgpu.conf`:

```
KERNEL_CMDLINE[default]+=" apple_gmux.force_igd=1"
```

Do not blacklist amdgpu. Do not add `enable_guc=3`. Rebuild
the UKI only on `@` (`limine-mkinitcpio`), never from a Snapper overlay.

## Delta from install default

Stock startup (this model): gmux → AMD; i915 is present but does not drive
the LCD. Hyprland binds the 5300M and dies.

Committed startup: gmux → Intel; i915 drives `eDP-1` at 3072×1920.

| Knob | Install default | This decision |
|---|---|---|
| Internal LCD owner | AMD `0000:03:00.0` | Intel `0000:00:02.0` |
| `/etc/limine-entry-tool.d/z-t2-amdgpu.conf` | empty (no gmux token) | `KERNEL_CMDLINE[default]+=" apple_gmux.force_igd=1"` |
| UKI cmdline vs install | (no `force_igd`) | adds `apple_gmux.force_igd=1` |

Plymouth hook, `quiet splash`, and `plymouth.enable` unset are **install
defaults again**. They are not part of this delta. Journald is stock `5m`.

## Evidence

3/3 unedited Limine entry-1 `@` boots with `force_igd` and stock journald:
graphical session, eDP-1 → `0000:00:02.0`, no SMU / GPU reset / sdma
timeout. Same kernel, mesa, and firmware as 5/5 SMU deaths with the panel
on AMD.

## Permanence

`z-t2-amdgpu.conf` is the committed file. Reboot keeps it. A later
`limine-mkinitcpio` on `@` must leave that file in place or the mux
returns to AMD and the hang returns.

## Consequences

Desktop scanout is UHD 630, not the 5300M. Limine `default_entry` is
unchanged.

## Rejected

- Omarchy PR 5129 as written.
- `i915.enable_guc=3`.
- Shipping a journald `SyncIntervalSec` other than stock.
