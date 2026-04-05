# Testing the mt76 rx_rate patch

Step by step instructions for applying the patch, building the modules, loading them into a running kernel, and verifying the new `rx bitrate:` line shows up in `iw station dump`. About 15 minutes. No reboot required. Fully reversible.

## What you need

- A Linux machine with a MediaTek Wi-Fi 6 adapter:
  - Internal: any laptop with an MT7921, MT7922, or MT7925 chipset (Framework, recent ThinkPad, HP ProBook, Surface, Lenovo Yoga, many Asus Zenbook / VivoBook, some Chromebooks)
  - USB: an Alfa AWUS036AXML, or any MT7921U dongle (there are many)
- Matching kernel headers or `kernel-devel` package
  - Fedora: `sudo dnf install kernel-devel`
  - Debian / Ubuntu: `sudo apt install linux-headers-$(uname -r)`
  - Arch: `sudo pacman -S linux-headers`
- Build toolchain (usually already installed on any machine that compiles anything):
  - Fedora: `sudo dnf install gcc make bison flex elfutils-libelf-devel openssl-devel bc dwarves rsync perl`
  - Debian / Ubuntu: `sudo apt install build-essential libelf-dev libssl-dev bison flex bc dwarves rsync`
- Secure Boot **disabled**, or a signing workflow set up. Unsigned out-of-tree modules will not load with Secure Boot on. Check with `sudo mokutil --sb-state`.
- Internet access (you're going to download a small part of the Linux kernel source).

## Step 1: note your running kernel version

```
uname -r
```

Example output: `6.19.10-200.fc43.x86_64`. The matching vanilla upstream version is the first part before the distro suffix. For Fedora `6.19.10-200.fc43.x86_64`, that's Linux `6.19.10`. For Debian `6.12.9-amd64`, that's Linux `6.12.9`. For Arch rolling, check `pacman -Q linux`.

## Step 2: get the matching mt76 source subtree

You only need the `mt76` subdirectory, not the whole kernel. About 3.5 MB.

```
mkdir -p ~/mt76-rxrate && cd ~/mt76-rxrate

# Replace 6.19.10 with your running kernel's version
curl -LO https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.19.10.tar.xz

tar -xJf linux-6.19.10.tar.xz --strip-components=5 \
    linux-6.19.10/drivers/net/wireless/mediatek/mt76/

ls mt76 | head        # should show agg-rx.c, channel.c, mt7921/, mt7925/, ...
```

## Step 3: apply the patch

Grab the patch file from this repo:

```
curl -LO https://raw.githubusercontent.com/Lucid-Duck/mt76-rxrate/main/0001-wifi-mt76-mt792x-report-last-RX-rate-in-station-stat.patch
```

Apply it to the mt76 subtree. The patch uses git-format paths like `a/drivers/net/wireless/mediatek/mt76/mt7921/mac.c`, and your working directory is the extracted `mt76/` dir, so `patch` needs to strip six leading components (`a/`, `drivers/`, `net/`, `wireless/`, `mediatek/`, `mt76/`) to end up at `mt7921/mac.c`:

```
cd mt76
patch -p6 < ../0001-wifi-mt76-mt792x-report-last-RX-rate-in-station-stat.patch
```

Expected output: seven files patched, no offsets or rejects.

## Step 4: build the modules

```
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules -j$(nproc)
```

Should complete in a few minutes with no errors or warnings. You'll see "Skipping BTF generation for ... due to unavailability of vmlinux" which is informational and harmless.

When done, you have about 25 freshly built `.ko` files scattered under the current directory and its `mt7921/`, `mt7925/`, etc subdirs.

## Step 5: hot-swap stock modules for the patched ones

If your machine has a second network path (ethernet, another Wi-Fi adapter, USB tether), the brief unload will be invisible. If the MediaTek adapter is your only network access, you'll briefly lose connection and reacquire it after reload.

**Before unloading**, if you're using any userspace that holds the interface (a running iperf3, a tcpdump, etc.), stop it first.

Unload stock modules in dependency order:

```
sudo ip link set wlan0 down    # substitute your actual interface name
for m in mt7921u mt7921e mt7921s mt7925u mt7925e \
         mt792x_usb mt792x_lib \
         mt7921_common mt7925_common \
         mt76_connac_lib mt76_usb mt76; do
    sudo rmmod $m 2>/dev/null
done
lsmod | grep -E "mt7|mt76"   # should be empty
```

Load the patched modules in reverse order. Pick the set that matches your hardware:

**For USB MT7921 (Alfa and similar):**
```
sudo insmod ./mt76.ko
sudo insmod ./mt76-connac-lib.ko
sudo insmod ./mt76-usb.ko
sudo insmod ./mt792x-lib.ko
sudo insmod ./mt792x-usb.ko
sudo insmod ./mt7921/mt7921-common.ko
sudo insmod ./mt7921/mt7921u.ko
```

**For PCIe MT7921 (most laptop-embedded):**
```
sudo insmod ./mt76.ko
sudo insmod ./mt76-connac-lib.ko
sudo insmod ./mt792x-lib.ko
sudo insmod ./mt7921/mt7921-common.ko
sudo insmod ./mt7921/mt7921e.ko
```

**For USB MT7925:**
```
sudo insmod ./mt76.ko
sudo insmod ./mt76-connac-lib.ko
sudo insmod ./mt76-usb.ko
sudo insmod ./mt792x-lib.ko
sudo insmod ./mt792x-usb.ko
sudo insmod ./mt7925/mt7925-common.ko
sudo insmod ./mt7925/mt7925u.ko
```

**For PCIe MT7925:**
```
sudo insmod ./mt76.ko
sudo insmod ./mt76-connac-lib.ko
sudo insmod ./mt792x-lib.ko
sudo insmod ./mt7925/mt7925-common.ko
sudo insmod ./mt7925/mt7925e.ko
```

Check dmesg for any errors:
```
sudo dmesg | tail -20
```

You should see the usual firmware load messages, the adapter re-enumerating, and the interface renaming. You'll also see a line that looks like this:

```
mt76: loading out-of-tree module taints kernel.
mt76: module verification failed: signature and/or required key missing - tainting kernel
```

That's expected and harmless. It just means the modules aren't signed with your distro's signing key (because we built them locally). The "tainted kernel" flag stays set until you reboot, but it has no functional effect.

## Step 6: reconnect and verify the new rx bitrate line

Let NetworkManager reassociate (it should do so automatically after the adapter comes back), or do it manually with `wpa_supplicant` if you prefer. Then:

```
iw dev wlan0 station dump | grep bitrate
```

**You should see BOTH lines**:

```
        tx bitrate:     1200.9 MBit/s 80MHz HE-MCS 11 HE-NSS 2 HE-GI 0 HE-DCM 0
        rx bitrate:     1200.9 MBit/s 80MHz HE-MCS 11 HE-NSS 2 HE-GI 0 HE-DCM 0
```

Before the patch there would only be a `tx bitrate:` line.

Immediately after association, before any traffic flows, the rx rate will show the basic rate of the current band (1.0 Mbit/s for 2.4 GHz CCK, 6.0 Mbit/s for 5/6 GHz OFDM). That's correct because beacons and management frames are sent at the basic rate, and those are the only frames arriving until actual data traffic starts.

## Step 7: watch it track rate control in real time

Generate some real traffic (replace the URL with anything that serves a big blob):

```
curl -o /dev/null https://speed.cloudflare.com/__down?bytes=2000000000
```

In another terminal:

```
watch -n 0.5 "iw dev wlan0 station dump | grep bitrate"
```

You should see the RX rate climb from the basic rate up to whatever HE-MCS your AP negotiates, and adapt frame-by-frame as the link conditions shift. In strong signal conditions with a Wi-Fi 6 AP, expect to see HE-MCS 11 NSS 2 (the maximum for 2x2 HE). If you move the laptop around you should see the rate actively adjust.

## Step 8: record what you saw

Please add your result to [RESULTS.md](RESULTS.md) via a pull request, or just open an issue on this repo with the details. Template:

```
- Hardware: (exact chipset and adapter model, e.g. "MT7921E on Framework Laptop 13 AMD")
- Bus: (PCIe / USB 2.0 / USB 3.0)
- Kernel: (output of `uname -r`)
- Distro: (Fedora 43, Ubuntu 24.04, Arch rolling, etc.)
- AP: (brand/model if known, or just "unknown consumer AP")
- Bands tested: (2.4 / 5 / 6 GHz, which of them)
- Result: works / partial / broken
- Peak rx bitrate observed: (e.g. "1200.9 Mbit/s HE-MCS 11 NSS 2 80 MHz")
- Anything unusual in dmesg: (or "clean")
```

A simple "worked on my Framework 13, 6 GHz, HE-MCS 11, clean dmesg" is enough. Detailed reports are great but not required.

## Rolling back to stock

Whenever you want to go back to the distro-packaged modules:

```
sudo ip link set wlan0 down
for m in mt7921u mt7921e mt7925u mt7925e \
         mt792x_usb mt792x_lib \
         mt7921_common mt7925_common \
         mt76_connac_lib mt76_usb mt76; do
    sudo rmmod $m 2>/dev/null
done
sudo modprobe mt7921u   # or mt7921e / mt7925u / mt7925e as appropriate
```

That loads the stock distro-signed modules again. The kernel taint flag persists until you reboot but does nothing.

If anything gets stuck and a module refuses to unload, a regular reboot always returns to stock because the patched modules were never installed to `/lib/modules/...`, only held in memory.

## Common gotchas

- **"module verification failed"** -- expected. Unsigned out-of-tree modules, not a problem.
- **"invalid module format"** -- your modules were built against a different kernel version. Check `modinfo ./mt792x-lib.ko | grep vermagic` vs `uname -r`. If they don't match, rebuild against the right source.
- **Association fails after reload** -- usually NetworkManager racing with autoconnect. Try `sudo systemctl restart NetworkManager`.
- **patch fails with "can't find file to patch"** -- make sure you ran `patch -p6` (not `-p1`) and your working directory is the extracted `mt76/` directory, not its parent.
- **No `rx bitrate` line even after the patch** -- confirm the patched modules are actually loaded: `modinfo mt792x-lib | grep filename` should point to your local `.ko`, not `/lib/modules/...`.

## Questions or weird results

Open an issue on this repo and include:
- The `modinfo mt792x-lib` output
- The `iw dev wlan0 station dump` output (before and after the patch if possible)
- The last 20 lines of `dmesg`
- Hardware and kernel version
