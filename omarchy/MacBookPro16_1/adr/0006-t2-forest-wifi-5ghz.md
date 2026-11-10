# ADR 0006: Pin forest Wi-Fi to 5 GHz

- Date: 2026-11-09
- Status: Accepted; verified in the current session
- Machine: MacBookPro16,1 (T2)
- Wi-Fi: Broadcom BCM4364 802.11ac [`14e4:4464`] (rev 04), Apple subsystem [`106b:07bf`]
- Interface: `wlp5s0` (`88:66:5a:43:0d:16`), driver `brcmfmac`
- Software: Omarchy 4.5.0-3, linux-t2 7.2.11.arch1-1, NetworkManager 1.58.1-2

## Context

`wlp5s0` was associated to SSID `forest` with a working default route, and
throughput was low.

The access point advertises `forest` on two radios:

| Radio | BSSID | Channel | Client channel width | Signal |
| --- | --- | --- | --- | --- |
| 2.4 GHz | `10:74:C5:88:9C:8E` | 11 (2462 MHz) | 20 MHz | about −56 dBm |
| 5 GHz | `10:74:C5:88:9C:8F` | 100 (5500 MHz) | 80 MHz once associated | about −69 dBm |

Both beacons report a maximum rate of 1170 Mbit/s. That figure is the AP's
Wi-Fi 6 rate. This card is 802.11ac.

On the 2.4 GHz radio the link stayed at 144.4 Mbit/s in each direction, the
one-stream 802.11n 20 MHz short-guard rate. A 10 MB download from Cloudflare
measured about 27 Mbit/s. The gateway `192.168.1.1` answered with no loss at
about 2 ms. A 20-packet ping to `1.1.1.1` lost one packet at about 21 ms.
DNS resolution completed immediately. `iw` reported power save off.

Channel 11 overlaps neighboring 2.4 GHz networks. The 5 GHz `forest` radio is
on DFS channel 100. `iw reg get` showed the global domain as `US` and phy#0 as
country `99` (`DFS-UNSET`). The NetworkManager profile had no band, channel, or
BSSID pin. `802-11-wireless.seen-bssids` already listed both radios, so the
client had used 5 GHz and then remained on 2.4 GHz. That association was about
19 hours old.

This boot logged one failure, at 00:32. The supplicant left `completed`, the
link timed out, and activation of `forest` failed with `ssid-not-found`. The
profile reconnected at 00:35 and stayed up until the change below.

## Decision

Pin the `forest` profile to the 5 GHz band. The profile UUID is
`2748701b-8ebf-429f-91a6-f803a70e6a84`.

```bash
nmcli connection modify forest 802-11-wireless.band a
nmcli connection down forest
nmcli connection up forest
```

The setting is stored in that NetworkManager connection. It does not apply to
other SSIDs and it is not a kernel parameter.

To return to automatic band selection:

```bash
nmcli connection modify forest 802-11-wireless.band ""
nmcli connection up forest
```

## Evidence and verification

After the pin, `wlp5s0` associated to `10:74:c5:88:9c:8f`:

- Channel 100, 80 MHz, center 5530 MHz
- Signal −69 dBm at association, −71 dBm a few minutes later
- Receive rate 650 Mbit/s, transmit rate 390 Mbit/s
- No transmit failures on the new association
- Address still `192.168.1.168/24`, default route via `192.168.1.1` on `wlp5s0`

Checks run immediately after activation:

```text
ping 192.168.1.1: 5/5, average 2.9 ms
ping 1.1.1.1:     8/8, average 19.9 ms
20 MB from speed.cloudflare.com: 11.6 MB/s (about 93 Mbit/s)
```

The association stayed up through the download. This session shows that
`brcmfmac` will associate to DFS channel 100 and pass traffic while phy#0
remains country `99`. It does not show that the link survives a radar event
or a later roam.

Useful checks:

```bash
nmcli -g 802-11-wireless.band connection show forest   # a
iw dev wlp5s0 info                                     # channel 100, 80 MHz
iw dev wlp5s0 link                                     # 5500 MHz, hundreds of Mbit/s
```

## Consequences

- `forest` will not use the 2.4 GHz radio while this pin is set. If the 5 GHz
  radio disappears, the connection fails instead of falling back.
- Throughput in this session rose from about 27 Mbit/s to about 93 Mbit/s.
- Moving the AP's 5 GHz radio to a non-DFS channel in 36–48 would remove the
  DFS dependency. The band pin can stay either way. It is what keeps this card
  off the 20 MHz 2.4 GHz radio.
