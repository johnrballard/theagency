# ADR 0004: Preserve USB-C hotplug with Titan Ridge runtime power management

- Date: 2026-08-29
- Status: Accepted
- Machine: MacBookPro16,1 (T2)
- USB controllers: Intel JHL7540 Titan Ridge 4C (`8086:15ec`)
- Software: Omarchy 4.0.1, linux-t2 7.1.8

## Context

USB storage connected directly to the MacBook's USB-C ports received power but
did not appear in the USB or block-device inventories. The same drive worked
through a powered Thunderbolt dock. Another directly connected USB storage
device also powered on without producing a kernel USB-connect event.

The two onboard Titan Ridge xHCI controllers are:

```text
0000:09:00.0  Intel JHL7540 xHCI  8086:15ec
0000:7f:00.0  Intel JHL7540 xHCI  8086:15ec
```

The top-left USB-C port was observed below `0000:09:00.0` as USB path `4-2`.
When unused, both PCI controllers used `power/control=auto` and reached runtime
suspend. Connecting a drive did not resume the relevant controller or generate
a USB-connect event.

The dock succeeds through a different topology. Thunderbolt establishes a PCIe
tunnel to xHCI controllers in the dock, so its downstream USB devices do not
depend on the Mac's suspended onboard xHCI controller. External power helps the
dock and its peripherals remain powered, but power alone does not explain the
difference in enumeration paths.

## Decision

Retain automatic runtime power management for both onboard Titan Ridge USB
controllers, but prevent them from entering PCI D3cold. Enable wakeup for the
controllers and their USB root hubs.

Install `/etc/udev/rules.d/99-t2-titan-ridge-usb-pm.rules`:

```udev
# MacBookPro16,1: Titan Ridge USB fails to signal hotplug after entering D3cold.
# Retain runtime power management, but restrict these controllers to a wakeable state.
ACTION=="add|bind", SUBSYSTEM=="pci", ATTR{vendor}=="0x8086", ATTR{device}=="0x15ec", ATTR{d3cold_allowed}="0", ATTR{power/wakeup}="enabled", ATTR{power/control}="auto"

# Arm only the USB root hubs below the affected Titan Ridge controllers for wakeup.
ACTION=="add", SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{devnum}=="1", ATTRS{vendor}=="0x8086", ATTRS{device}=="0x15ec", ATTR{power/wakeup}="enabled"
```

This is narrower than forcing `power/control=on`: unused controllers can still
runtime-suspend, but Linux may not remove power from them completely.

## Evidence and verification

The diagnosis was tested in stages:

1. Default `auto`, with D3cold allowed: the directly attached drive received
   power, but the controller remained suspended and no connection event
   appeared.
2. `power/control=on`: the same port immediately enumerated the USB storage
   device, proving that the cable, port, xHCI driver, and drive could work.
3. `auto`, D3cold allowed, and USB root-hub wakeup enabled: disconnect was
   logged, but reconnect was missed and both controllers remained suspended.
4. `auto`, `d3cold_allowed=0`, and wakeup enabled: reconnect generated a new
   SuperSpeed Plus event and exposed the storage device as a block device.
5. After installing and triggering the persistent rule, another connection
   enumerated the device successfully while the affected controller stayed on
   `auto`; the unused controller was runtime-suspended.

The A/B tests isolate the failure to the transition into or wake from D3cold.
They do not establish whether the lowest-level defect is in Apple/T2 firmware,
Titan Ridge wake signaling, or the Linux PCI/xHCI integration. No upstream
quirk specific to `8086:15ec` was identified during this investigation.

Useful checks after reboot or a kernel update:

```bash
cat /sys/bus/pci/devices/0000:09:00.0/power/control
cat /sys/bus/pci/devices/0000:09:00.0/d3cold_allowed
cat /sys/bus/pci/devices/0000:09:00.0/power/wakeup
cat /sys/bus/pci/devices/0000:7f:00.0/power/runtime_status
lsblk -o NAME,MODEL,TRAN,SIZE,FSTYPE,MOUNTPOINTS
journalctl -b -k | grep -E 'xhci|new (high|SuperSpeed)|USB disconnect'
```

The expected persistent values are `control=auto`, `d3cold_allowed=0`, and
`wakeup=enabled`. An unused controller may report `runtime_status=suspended`;
that is expected and confirms that automatic power management remains active.

## Consequences

- Direct USB-C hotplug works without forcing the controllers permanently on.
- Runtime suspend remains available, but the two Titan Ridge controllers cannot
  use the deepest PCI power-off state.
- Idle power use may be slightly higher than the default D3cold policy and
  should be observed during normal battery use.
- Enabling root-hub wake may permit USB activity on these controllers to wake
  them when suspended.
- A future kernel or firmware fix may make this local rule unnecessary.

## Rollback

Remove the rule, reload udev, and reboot:

```bash
sudo rm /etc/udev/rules.d/99-t2-titan-ridge-usb-pm.rules
sudo udevadm control --reload-rules
sudo reboot
```

After rollback, the default policy permits D3cold again. If direct USB-C
hotplug then fails, temporarily forcing both controllers to `on` remains a
diagnostic or emergency workaround, not the accepted configuration.

## References

- [Linux PCI power management](https://github.com/torvalds/linux/blob/master/Documentation/power/pci.rst)
- [Linux USB power management](https://github.com/torvalds/linux/blob/master/Documentation/driver-api/usb/power-management.rst)
- [Linux PCI sysfs ABI, including `d3cold_allowed`](https://github.com/torvalds/linux/blob/master/Documentation/ABI/testing/sysfs-bus-pci)
- [Linux xHCI PCI driver](https://github.com/torvalds/linux/blob/master/drivers/usb/host/xhci-pci.c)

## Related decisions

- [ADR 0002: T2 Intel/AMD hybrid graphics configuration](0002-t2-intel-amd-hybrid-graphics.md)
- [ADR 0003: Stabilize LG UltraFine hotplug on T2 Navi 14](0003-t2-lg-ultrafine-hotplug-stability.md)
