# FPGA Image Update Guide

The card's "firmware" is a PolarFire FPGA image. It is programmed over JTAG
with a Microchip FlashPro5 or FlashPro6 programmer and the free FlashPro
Express software. The FPGA is non-volatile and there is no configuration
flash on the board, so an interrupted or failed update never bricks the
card: reprogram and power-cycle.

## What you need

- FlashPro5/FlashPro6 programmer and a 10-pin 1.27 mm (0.050") ribbon
- FlashPro Express (Windows or Linux, part of the free Libero SoC download)
- `quantarng-pcie-fw-<version>.zip` from the
  [Releases page](https://github.com/dyber-pqc/QuantaRNG-PCIe-Pub/releases/latest)

## 1. Verify the image

```bash
sha256sum -c quantarng-pcie-fw-<version>.zip.sha256
gpg --verify quantarng-pcie-fw-<version>.zip.asc quantarng-pcie-fw-<version>.zip
unzip quantarng-pcie-fw-<version>.zip
sha256sum -c quantarng-pcie-fw-<version>.sha256      # inner checksums of the .job
```

The signing key fingerprint is published at
[dyber-pqc.com/keys](https://dyber-pqc.com/keys). Do not program an image
whose signature does not verify.

## 2. Connect

1. Unload the driver: `sudo modprobe -r quantarng_pcie`. The card can stay
   in the slot and powered; or program it on the bench with 12 V applied to
   the edge connector.
2. Plug the FlashPro ribbon into the JTAG header **J701** (keyed, near the
   FPGA). The header supplies the 2.5 V reference the programmer needs.

## 3. Program

1. FlashPro Express → *New Job Project from FlashPro Express Job* → select
   `quantarng-pcie-fw-<version>.job`.
2. The programmer and device (`MPF300TS`) appear; action *PROGRAM* → *Run*.
   Programming takes about 90 seconds; wait for *PASSED*.
3. Power-cycle the host (a warm reboot may not reload the fabric).

## 4. Confirm

```bash
sudo modprobe quantarng_pcie
dmesg | grep quantarng          # reports the RTL revision
qrngpcie-ctl info               # ID magic QRNG, ready 1
```

## Rollback

Program the previous release's `.job` the same way. All released images
are kept on the Releases page.

## Driver compatibility

Each release notes the minimum driver version for its image. The driver
checks the card's ID register at load and refuses to bind to an image it
does not understand rather than delivering misdecoded data.
