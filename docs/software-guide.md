# Software Guide

## Reading entropy

### From the kernel pool (recommended for most applications)

Once the driver is loaded the card is the kernel's current hardware RNG
(`/sys/class/misc/hw_random/rng_current`). Every consumer of `/dev/random`,
`/dev/urandom` and `getrandom(2)` benefits without changes. The driver
credits the pool with full entropy for conditioned output (`hwrng_quality`)
and stops feeding it the instant a health test fails.

### Directly from the device

`/dev/quantarng-pcieN` is a plain character device:

```c
int fd = open("/dev/quantarng-pcie0", O_RDONLY);
unsigned char key[32];
ssize_t n = read(fd, key, sizeof key);      /* n == 32, or -1 with errno EIO on a health alarm */
```

```python
with open("/dev/quantarng-pcie0", "rb", buffering=0) as f:
    key = f.read(32)
```

```bash
head -c 1048576 /dev/quantarng-pcie0 > random.bin
```

Reads block briefly if the on-card FIFO is momentarily empty (rare: the
conditioner outruns the host) and return `EIO` while the card reports a
health alarm or has not passed its startup tests. Any read size works.

The `libquantarng` C library and `quantarng` Python package used with
QuantaRNG USB gain a PCIe backend in the next release, so code written
against `qrng_open()` / `qrng_read_full()` will work unchanged with either
device.

## qrngpcie-ctl

The control tool talks to the driver's ioctl interface.

```bash
qrngpcie-ctl info                 # identity, PCIe link state, conditioner, counters, decoded STATUS
qrngpcie-ctl status -w 1          # live status refresh every second
qrngpcie-ctl health               # cutoffs, last variance, alarms
qrngpcie-ctl mcv                  # per-lane most-common-value min-entropy estimate
qrngpcie-ctl dump -n 1073741824 -o cond.bin        # 1 GiB conditioned output
qrngpcie-ctl dump --raw -n 67108864 -o raw.bin     # 64 MiB raw samples (exclusive)
qrngpcie-ctl bench -t 10          # sustained read throughput
qrngpcie-ctl marker               # event marker: latch the sample counter
```

Health telemetry also appears in sysfs:

```bash
ls /sys/class/quantarng/quantarng-pcie0/
# conditioner fifo_level gate_en health health_cfg id ltssm mcv raw_mode
# recalibrate reset_health sample_count stats status
```

## Raw capture for your own SP 800-90B assessment

Raw mode delivers the ADC samples before health gating and conditioning:
each 8-byte word is one simultaneous sample of the eight capture lanes,
byte *n* = lane *n*, offset binary (0x80 = mid-scale) unless `--twos`.
Raw mode is exclusive: it needs the only open handle on the device, it
unregisters the kernel hwrng while active, and it ends when the handle
closes. The host cannot keep up with 500 MS/s over programmed I/O, so raw
captures are decimated; the tool warns when the on-card FIFO overflowed and
the sample counter lets you measure gaps. Feed per-lane byte streams to
NIST's `ea_non_iid` to reproduce the min-entropy figures in your card's
report.

## Health-test alarms

The card runs the Repetition Count Test, the Adaptive Proportion Test
(window 512) and a variance test on every raw sample, in hardware. On an
alarm:

1. `STATUS.health_fail` sets and `ready` clears; with `gate_en=1` the
   conditioner stops taking input.
2. The driver refuses reads (`EIO`), discards the FIFO, stops the hwrng
   feed, and counts the event (`stats` in sysfs).
3. Once the cause is gone, `qrngpcie-ctl health reset` (or
   `echo 1 > /sys/class/quantarng/quantarng-pcie0/reset_health`) re-arms the startup tests; `ready` returns
   after 1024 clean samples.

Frequent alarms on a healthy card mean the cutoffs are tighter than the
source warrants; use the values from the card's assessment report.
