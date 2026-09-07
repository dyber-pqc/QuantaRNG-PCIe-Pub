# Driver Install Guide

QuantaRNG PCIe needs a small kernel module on Linux. It is delivered as
source (DKMS-ready) so it follows your kernel upgrades. Supported: 64-bit
Linux 5.10 or newer on x86-64 and arm64.

## 1. Install the card

Power off, seat the card in any x4, x8 or x16 PCIe slot, secure the bracket,
power on. Check the host sees it:

```bash
lspci -nn | grep -i 11aa
# 0000:03:00.0 Encryption controller [1080]: Microchip ... [11aa:1556]
sudo lspci -vv -s 03:00.0 | grep -E "LnkSta|Region 0"
#   Region 0: Memory at ... (64-bit, prefetchable) [size=64K]
#   LnkSta: Speed 5GT/s, Width x4
```

If nothing shows, see the [Troubleshooting Guide](troubleshooting-guide.md).

## 2. Build and install the driver

```bash
tar xzf quantarng-pcie-driver-<version>.tar.gz
cd quantarng-pcie-driver-<version>
sudo apt-get install build-essential dkms linux-headers-$(uname -r)   # Debian/Ubuntu
# sudo dnf install kernel-devel dkms gcc make                          # Fedora/RHEL
sudo make -C linux dkms-install
```

This registers `quantarng_pcie` with DKMS, builds it for every installed
kernel, installs the udev rule, and will rebuild automatically after kernel
updates. Without DKMS: `sudo make -C linux install`.

## 3. Permissions and loading

```bash
sudo groupadd -f quantarng
sudo usermod -aG quantarng $USER        # log out and in again
sudo modprobe quantarng_pcie
dmesg | grep quantarng
# quantarng-pcie 0000:03:00.0: QuantaRNG PCIe rev 2.1 at 0000:03:00.0, /dev/quantarng-pcie0, hwrng on, driver 0.1.0
```

The module loads automatically at boot once installed (PCI ID match).

## 4. Verify

```bash
qrngpcie-ctl info                                  # ready 1, health_fail 0, dclk_pll_lock 1
cat /sys/class/quantarng/quantarng-pcie0/health    # "ready"
cat /sys/class/misc/hw_random/rng_current          # quantarng-pcie0
head -c 16 /dev/quantarng-pcie0 | xxd
```

## 5. Options

Create `/etc/modprobe.d/quantarng-pcie.conf`:

| Option | Default | Meaning |
|---|---|---|
| `hwrng_quality=N` | 1024 | Entropy credited to the kernel pool per 1024 bits. `0` mixes without credit |
| `hwrng_enable=0` | on | Do not register with the kernel hwrng framework |
| `gate_en=0` | on | Let the on-card conditioner keep running through alarms (observational mode; the driver still refuses reads) |
| `rct_cutoff`, `apt_cutoff`, `var_min`, `var_max` | card defaults | Health-test cutoffs. Change only to values from your assessment report |
| `adc_cal_on_probe=0` | on | Skip the ADC calibration at load |

Multiple cards enumerate as `/dev/quantarng-pcie0`, `1`, … in PCI order.

## 6. Uninstall

```bash
sudo modprobe -r quantarng_pcie
sudo make -C linux dkms-remove       # or: sudo make -C linux uninstall
```
