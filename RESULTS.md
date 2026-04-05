# Test Results

Community test results for the mt76 rx_rate patch. Add yours via pull request or by opening an issue.

## Template

```
- Hardware:
- Bus:
- Kernel:
- Distro:
- AP:
- Bands tested:
- Result:
- Peak rx bitrate observed:
- Anything unusual in dmesg:
```

## Confirmed working

### Lucid-Duck (patch author) -- 2026-04-04

- **Hardware:** Alfa AWUS036AXML (MT7921AU)
- **Bus:** USB 3.0 SuperSpeed
- **Kernel:** 6.19.10-200.fc43.x86_64
- **Distro:** Fedora 43
- **AP:** Tri-band Wi-Fi 6 consumer AP (2.4 / 5 / 6 GHz, WPA2-PSK + WPA3-SAE)
- **Bands tested:** 2.4 GHz ch 1 (20 MHz), 5 GHz ch 100 DFS (80 MHz), 6 GHz ch 5 (80 MHz)
- **Result:** Works on all three bands.
- **Peak rx bitrate observed:**
  - 2.4 GHz: 286.7 Mbit/s HE-MCS 11 NSS 2 (idle basic rate 1.0 Mbit/s CCK)
  - 5 GHz ch 100: 1200.9 Mbit/s HE-MCS 11 NSS 2 (idle basic rate 6.0 Mbit/s OFDM)
  - 6 GHz ch 5: 1200.9 Mbit/s HE-MCS 11 NSS 2 (idle basic rate 6.0 Mbit/s)
- **Stress tests:** 60s iperf3 at 715 Mbit/s with 300 rapid sta_dump samples caught 5 distinct rate values including HE-MCS 9/10/11 and both GI variants. 5x assoc/disassoc cycles, 3x module reload cycles, 5 concurrent sta_dump workers firing 250 queries. Zero warnings, zero crashes, zero races.
- **Anything unusual in dmesg:** Clean. Only the expected `loading out-of-tree module taints kernel` message.

## Waiting on

- MT7921 PCIe (laptop-embedded card)
- MT7922 any bus
- MT7925 any bus (Wi-Fi 7 sibling)
- Unusual MT7921 USB SKUs with non-standard VID:PIDs
- Low-signal / poor link conditions (rate below HE-MCS 11)
- HT and VHT encoding paths (hard to hit with a Wi-Fi 6 AP that negotiates HE)
