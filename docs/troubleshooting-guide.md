# Troubleshooting Guide

Collect these before contacting support: `lspci -vv -s <bdf>`,
`dmesg | grep -i -E "quantarng|pcieport"`, `qrngpcie-ctl info`, and the card
serial number.

## Card not detected (`lspci` shows nothing)

| Check | Action |
|---|---|
| Slot | Must be an open-ended or x4/x8/x16 slot with the card fully seated; try another slot |
| Power | The green LED near the bracket lights when the FPGA is configured. No LED: slot 12 V or card fault |
| BIOS | Enable "Above 4G decoding" / 64-bit BAR support; disable any "PCIe device whitelist" |
| Presence | Some slots require the PRSNT2# pin; use a different slot if the board never enumerates |

## Link is x1 or Gen1 (`LnkSta` not `5GT/s, Width x4`)

The card negotiates PCIe 2.0 x4. A x1 link still works, at lower PIO
throughput. If the link is x1 in a x4-capable slot, try a slot wired
directly to the CPU rather than the chipset; some root ports do not support
lane reversal, which this card relies on.

## Driver loads but reports `bad ID register 0xffffffffffffffff`

BAR0 is not responding: the link dropped after enumeration or the FPGA lost
its clock. Reseat the card; check `dmesg` for AER messages; power-cycle
(a warm reboot does not re-configure the FPGA if the 12 V rail stayed up).

## `qrngpcie-ctl info` shows `dclk_pll_lock 0` or `dclk_alive 0`

The ADC's data clock is not reaching the FPGA. Power-cycle fully. If it
persists the card needs service.

## Reads return `EIO` / `health_fail 1`

A health test tripped. `qrngpcie-ctl health` shows which one (rct, apt,
qcnr) and `qrngpcie-ctl mcv` shows per-lane min-entropy. Then:

```bash
qrngpcie-ctl calibrate      # re-run the ADC calibration
qrngpcie-ctl health reset   # re-arm the startup tests
```

Repeated alarms on one lane (`h_min` near 0 or `max_count` near the
window size) indicate a stuck lane: contact support. Alarms across all
lanes right after a thermal change usually clear after recalibration.

## Reads return `EBUSY` or `EPERM`

Another process holds the device in raw mode (`raw_mode` in sysfs = 1).
Raw mode is exclusive; wait for that capture to finish. Setting health
cutoffs and calibration constants needs `CAP_SYS_ADMIN`.

## Reads return `EAGAIN` / `ETIMEDOUT`

The FIFO stayed empty for a second: capture is not running. `qrngpcie-ctl
info` should show `capture 1`; if not, reload the driver.

## `rng_current` is not `quantarng-pcie0`

Another hwrng (`tpm-rng`, `virtio_rng`) has higher priority, or the driver
was loaded with `hwrng_enable=0`. Select it explicitly:

```bash
echo quantarng-pcie0 | sudo tee /sys/class/misc/hw_random/rng_current
```

## Throughput is only a few MB/s

Expected: the current hardware revision delivers through programmed I/O,
one 64-bit PCIe read per word. The on-card conditioner runs at ~0.9 Gbit/s
but the host can pull roughly 5–10 MB/s depending on the root complex.
This is ample for the kernel pool and key generation; a DMA path arrives
with the next hardware revision.

## Permission denied on `/dev/quantarng-pcie0`

Add yourself to the `quantarng` group (`sudo usermod -aG quantarng $USER`),
log in again, and confirm the udev rule is installed
(`/etc/udev/rules.d/99-quantarng-pcie.rules`).

## Secure Boot

DKMS-built modules must be signed with your MOK on Secure Boot systems
(`sudo dkms` prompts for enrolment on Ubuntu; see your distribution's
DKMS + Secure Boot instructions). `modprobe: Key was rejected by service`
is the symptom.
