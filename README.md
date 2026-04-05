# mt76-rxrate

A small patch for the Linux `mt76` wireless driver that makes MediaTek Wi-Fi 6 devices finally report the RX bitrate to `iw dev wlan0 station dump` and friends.

Looking for community testers before sending this upstream.

## The bug, in one paragraph

If you run `iw dev wlan0 station dump` on a Linux laptop with a MediaTek MT7921, MT7922, or MT7925 Wi-Fi chipset, you get a `tx bitrate:` line but no matching `rx bitrate:` line. It has been that way for years: the mt7921 driver landed in Linux around early 2021, the mt7925 driver in late 2023, and neither has ever populated `NL80211_STA_INFO_RX_BITRATE` in `sta_statistics`. The RX rate isn't broken in the radio, it just isn't being surfaced to user space. Dead function prototypes (`mt7921_mcu_get_rx_rate`, `mt7925_mcu_get_rx_rate`) have been sitting in the header files unimplemented nearly the whole time, hinting at an MCU-query approach that was never finished. The `mt7915` sibling driver in the same mt76 family does populate this field (via its own MCU query path), but that path returns an error on the newer mt792x firmware so it was never ported. Nobody seems to have noticed because the feature silently degrades: you get no error, you just get a missing line.

## Before and after

Before (current mainline, stock Fedora / Arch / Ubuntu kernels):

```
$ iw dev wlan0 station dump
Station 00:00:00:00:00:A2 (on wlan0)
        ...
        signal:         -29 dBm
        signal avg:     -29 dBm
        tx bitrate:     1200.9 MBit/s 80MHz HE-MCS 11 HE-NSS 2 HE-GI 0 HE-DCM 0
        tx duration:    14786 us
        last ack signal:-28 dBm
        ...
```

After, with this patch:

```
$ iw dev wlan0 station dump
Station 00:00:00:00:00:A2 (on wlan0)
        ...
        signal:         -29 dBm
        signal avg:     -29 dBm
        tx bitrate:     1200.9 MBit/s 80MHz HE-MCS 11 HE-NSS 2 HE-GI 0 HE-DCM 0
        tx duration:    14786 us
        rx bitrate:     1200.9 MBit/s 80MHz HE-MCS 11 HE-NSS 2 HE-GI 0 HE-DCM 0
        last ack signal:-28 dBm
        ...
```

That's the entire user-visible change. One new line.

## What it is not

- **Not a throughput improvement.** The radio already receives at the correct rate. The patch just makes the driver tell user space what that rate is.
- **Not a power saving.** Unchanged.
- **Not a security fix.** No CVE.
- **Not a new feature.** `NL80211_STA_INFO_RX_BITRATE` is a standard nl80211 field that has existed for years and is populated by several other Linux wireless drivers (including the `mt7915` sibling in the same mt76 family). It just never got wired up on the mt792x side.

## Who benefits

Anyone who reads link rate from nl80211:

- `iw dev wlan0 station dump` and `iw dev wlan0 link`
- `wavemon` (the curses Wi-Fi monitor)
- GNOME and KDE network widgets in their detailed views
- NetworkManager D-Bus clients that poll link quality
- Any script or tool that parses `iw station dump` to diagnose connectivity
- Pentest tools like `kismet`, `hcxdumptool`, `airodump-ng` in some modes
- Anyone trying to figure out why their Wi-Fi feels slow and looking for where the ceiling actually is

The affected hardware population is sizeable: MT7921, MT7922, and MT7925 are common MediaTek Wi-Fi 6 / 6E / 7 chipsets used as OEM WLAN modules in budget-to-midrange laptops across most major vendors (some Framework, Lenovo, HP, Asus, and other SKUs ship them rather than an Intel AX card), plus USB adapters like the Alfa AWUS036AXML and various USB dongles built around the MT7921U reference design.

## How the patch works

The hardware already decodes rate information into every RX descriptor's PRXV group. The existing function `mt76_connac2_mac_fill_rx_rate()` (for mt7921) and `mt7925_mac_fill_rx_rate()` (for mt7925) call on every received frame to populate `struct mt76_rx_status` for radiotap and monitor-mode output. All the patch does is:

1. Add a `struct rate_info rx_rate` cache field to `struct mt792x_link_sta`.
2. Call a tiny new shared helper `mt792x_mac_update_rx_rate()` right after the existing `fill_rx_rate` step in both chipsets' RX paths, which translates the `mt76_rx_status` fields into a `rate_info` and stores the result.
3. Read that cache in the shared `mt792x_sta_statistics()` to populate `sinfo->rxrate`, set the `NL80211_STA_INFO_RX_BITRATE` bit in `sinfo->filled`, and done.
4. Remove two dead function prototypes (`mt7921_mcu_get_rx_rate`, `mt7925_mcu_get_rx_rate`) that have been sitting in the header files for years without a corresponding implementation.

No new MCU round-trips. No new firmware commands. No changes to the hot RX path beyond one struct copy and a small switch statement. The reader side is a single field access. Matches the same pattern `iwlwifi`, `ath11k`, and `mt7915` already use for RX rate reporting.

The total patch is 69 insertions, 4 deletions, 7 files touched.

## Hardware tested so far

| Chipset | Bus | Band | Channel | Result |
|---------|-----|------|---------|--------|
| MT7921AU (Alfa AWUS036AXML) | USB 3.0 | 2.4 GHz | ch 1, 20 MHz | Works: idle 1.0 Mbit/s CCK, load 286.7 Mbit/s HE-MCS 11 NSS 2 |
| MT7921AU (Alfa AWUS036AXML) | USB 3.0 | 5 GHz | ch 100 DFS, 80 MHz | Works: idle 6.0 Mbit/s OFDM, load 1200.9 Mbit/s HE-MCS 11 NSS 2 |
| MT7921AU (Alfa AWUS036AXML) | USB 3.0 | 6 GHz | ch 5, 80 MHz | Works: idle 6.0 Mbit/s, load 1200.9 Mbit/s HE-MCS 11 NSS 2 |

Stress tested: 60-second iperf3 at 715 Mbit/s with 300 concurrent `station dump` samples at 200 ms intervals caught 5 distinct rate values including real-time rate control decisions (HE-MCS 9/10/11 with both GI variants). 5x assoc/disassoc cycles, 3x module reload cycles, 5 concurrent station-dump workers firing 250 queries. Zero kernel warnings, zero crashes, zero races.

**Looking for confirmations on:**

- MT7921 PCIe laptop cards (I only have the USB variant)
- MT7922 (the slightly newer revision of mt7921)
- MT7925 (the Wi-Fi 7 / Wi-Fi 6E sibling)
- Any SKU that enumerates at an unusual VID:PID

Please add your results to [RESULTS.md](RESULTS.md) via a PR or open an issue.

## How to test

See [TESTING.md](TESTING.md) for step-by-step instructions. Takes about 15 minutes, no reboot required, fully reversible via `rmmod` + `modprobe`.

## License

The patch follows the existing mt76 driver license (BSD-3-Clause-Clear, from the files it touches). Apply it, redistribute it, submit it upstream, whatever you like.

## Questions

Open an issue on this repo, or ping `@Lucid-Duck` on the morrownr/USB-WiFi tester thread.
