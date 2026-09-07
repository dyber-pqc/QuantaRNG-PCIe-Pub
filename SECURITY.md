# Security Policy

## Supported versions

| Version | Supported |
|---|---|
| 1.x.y (FPGA image + driver) | ✅ once released |
| 0.x.y | pre-release |

## Reporting a vulnerability

QuantaRNG PCIe is intended for cryptographic use. **Please do not report
security vulnerabilities through public GitHub issues.**

Email **security@dyber-pqc.com** with a description, reproduction steps or
proof of concept, an impact assessment and any suggested fix. Our PGP key:
[https://dyber-pqc.com/.well-known/pgp-key.txt](https://dyber-pqc.com/.well-known/pgp-key.txt).

- Within 48 hours: acknowledgment
- Within 7 days: preliminary assessment
- Within 90 days: patch released or coordinated disclosure plan agreed

## Threat model

### In scope

- Statistical bias in conditioned output or delivery of under-conditioned data
- Health-test bypass, false-pass conditions, or delivery of data buffered before an alarm
- Driver: ioctl privilege boundaries, raw-mode exposure, kernel entropy over-crediting
- Malformed register access that stalls the card or the host
- FPGA image integrity (unsigned or tampered programming jobs)
- Physical tampering that compromises the entropy source

### Out of scope

- Attacks requiring physical destruction of the card
- Host OS issues unrelated to the driver
- Denial of service by removing the card or cutting power
- The Ethernet port (not functional in the current hardware revision)

## Hardening recommendations

- Keep the driver's default `gate_en=1` so alarms stop the conditioner on-card.
- Leave `hwrng_quality` at the shipped value, or lower it if you prefer the
  kernel to treat the card as a mixer rather than a credited source.
- Restrict `/dev/quantarng-pcie*` to a dedicated group (shipped udev rule: `quantarng`).
- Verify the FPGA image checksum and signature before programming.
- Re-run SP 800-90B tests on raw captures periodically.

## CVE history

None.
