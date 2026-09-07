# QuantaRNG PCIe

**True hardware random number generator on a PCIe card — quantum shot noise sampled at 500 MS/s, NIST SP 800-90B health-tested and AES-conditioned in FPGA fabric, delivered straight into your kernel's entropy pool.**

QuantaRNG PCIe is a half-height, half-length PCI Express card by [Dyber, Inc.](https://dyber-pqc.com) for servers, HSM appliances and workstations that need a high-assurance entropy source with no USB and no serial protocol in the path. A Microchip PolarFire FPGA captures two 250 MS/s channels of quantum noise, runs continuous health tests on every sample, conditions the stream with AES-128 CBC-MAC and presents the result as a memory-mapped FIFO. The Linux driver feeds it to `/dev/random` and to `/dev/quantarng-pcie0` for applications.

This repository hosts the **official software downloads and end-user documentation**.

---

## 📥 Downloads

Get the latest release from the **[Releases page](https://github.com/dyber-pqc/QuantaRNG-PCIe-Pub/releases/latest)**.

| Item | What to download |
|---|---|
| **Linux driver** (source, DKMS-ready) | `quantarng-pcie-driver-<version>.tar.gz` |
| **Control tool** | `qrngpcie-ctl-linux-x86_64`, `qrngpcie-ctl-linux-aarch64` (static binaries) |
| **FPGA image** | `quantarng-pcie-fw-<version>.zip` — FlashPro Express job + SHA-256 + GPG signature |
| **C / Python SDK** | coming with the next release (same `libquantarng` API as QuantaRNG USB) |
| **Windows** | driver and dashboard in development |

Every file ships with a `.sha256`; the FPGA image also carries a detached GPG signature. The signing key fingerprint is published at [dyber-pqc.com/keys](https://dyber-pqc.com/keys).

### Quick start (Linux)

```bash
tar xzf quantarng-pcie-driver-<version>.tar.gz && cd quantarng-pcie-driver-<version>
sudo apt-get install build-essential dkms linux-headers-$(uname -r)
sudo make -C linux dkms-install
sudo groupadd -f quantarng && sudo usermod -aG quantarng $USER    # then log in again
sudo modprobe quantarng_pcie

qrngpcie-ctl info                     # identity, link, health
head -c 1048576 /dev/quantarng-pcie0 > random.bin
cat /sys/class/misc/hw_random/rng_current   # -> quantarng-pcie0
```

Full instructions: [Driver Install Guide](docs/driver-install-guide.md).

---

## ✨ About the card

- **Quantum entropy source** — shot noise from reverse-biased semiconductor junctions, two independent channels
- **500 MS/s raw sampling** (2 × 250 MS/s, 8-bit) into a PolarFire FPGA
- **Continuous NIST SP 800-90B health tests in hardware** — Repetition Count, Adaptive Proportion and a variance test on every sample, with per-lane min-entropy telemetry you can read at any time
- **AES-128 CBC-MAC conditioning** (NIST-vetted construction), ~0.9 Gbit/s on-card
- **Fail-closed delivery** — no data leaves the card before its startup tests pass, none during an alarm, and buffered data from before an alarm is discarded
- **Raw tap for your own assessment** — capture unconditioned samples for SP 800-90B testing; the driver guarantees raw samples never reach the kernel pool
- **PCIe 2.0 x4**, x16 mechanical, half-height half-length, 12 V slot-powered, no cables
- **No firmware to trust on the card** — there is no CPU; the design is a fixed fabric image you can verify by checksum and signature

Rev 2 delivers entropy to the host through programmed I/O, which caps sustained throughput at roughly 5–10 MB/s per card — far more than any kernel pool needs, and enough for most application use. A DMA engine for full-rate delivery is planned for the next hardware revision.

---

## 📚 Documentation

| Guide | Contents |
|---|---|
| [**Driver Install Guide**](docs/driver-install-guide.md) | DKMS install, permissions, module options, sysfs, verifying the kernel pool is fed |
| [Software Guide](docs/software-guide.md) | Reading entropy from applications, `qrngpcie-ctl`, raw capture, health telemetry |
| [Troubleshooting Guide](docs/troubleshooting-guide.md) | Card not detected, link width, health alarms, read errors |
| [FPGA Image Update Guide](docs/fpga-image-update-guide.md) | Verifying and programming a new image over JTAG |

---

## 🔐 Verifying downloads

```bash
sha256sum -c quantarng-pcie-fw-<version>.zip.sha256
gpg --verify quantarng-pcie-fw-<version>.zip.asc quantarng-pcie-fw-<version>.zip
```

---

## 🆘 Support

- Start with the [Troubleshooting Guide](docs/troubleshooting-guide.md)
- **support@dyber.org** — include the card serial number, `qrngpcie-ctl info` output and `dmesg | grep quantarng`
- **security@dyber-pqc.com** — for security reports (see [SECURITY.md](SECURITY.md); please don't open public issues for vulnerabilities)

---

## 📄 License

Software in this repository and in the releases is licensed under the [MIT License](LICENSE). The FPGA image is provided for use with QuantaRNG hardware only.

---

**Made by [Dyber, Inc.](https://dyber-pqc.com)** — post-quantum cryptography, classical entropy, real silicon.
