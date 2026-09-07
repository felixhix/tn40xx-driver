# Delock 89475 on Proxmox VE 9

This branch tracks support and testing for the single-port Delock 89475
10 GbE SFP+ adapter on Proxmox VE 9.

## Hardware

- Adapter: Delock 89475
- Board marking: `PN64010-5X2A`
- MAC: Tehuti TN4010B0
- PHY: Texas Instruments TLK10031
- PCI device: `1fc9:4025`
- PCI subsystem: `1fc9:3015`

The vendor driver identifies this PHY through its TLK10232 support path. A
specific MAC address, board serial number, and photographs are intentionally
not included here.

## Observed failure with the in-tree driver

On Proxmox VE 9 with Linux `7.0.14-15-pve`, the in-tree `tn40xx` driver treats
this PCI ID as an Aquantia AQR105 design. Bringing the interface up then fails
while validating XAUI and attaching the PHY (`-EINVAL`). That hardware mapping
does not match this Delock board revision.

## Branch scope

The first change is deliberately small: it fixes PHY discovery when the MDIO
port number is zero and resets the PHY through the discovered MDIO port. This
is the same behavior that was used successfully with this card on the previous
Proxmox host.

Further changes should remain separate commits:

1. Linux 7.x / Proxmox VE 9 compatibility cleanup.
2. Short-frame TX accounting, after isolated review and testing.
3. DKMS packaging and installation documentation.

## Build result

The TI-PHY-only configuration (`TL=YES`) builds successfully against the
Proxmox kernel headers for `7.0.14-15-pve`. No Linux 7 API port was required.
The compatibility cleanup on this branch only removes duplicate `MIN` and
`MAX` definitions that otherwise produce compiler warnings with Linux 7 and
uses the current Kbuild `ccflags-y` variable. The latter is required so the
selected `PHY_TLK10232` feature define is present in the compiled module;
Linux 7 no longer honors the driver's old `EXTRA_CFLAGS` assignment here.

## Runtime result

On `2026-09-07`, the signed module was loaded successfully after enrolling the
host's DKMS Machine Owner Key. The driver reported:

- `PHY_TLK10232` enabled
- SVID PHY type `7`
- MDIO address `6`
- PHY ID `40005100`
- `Link Up 10G`

The link initially toggled while coming up, then remained at 10 Gb/s full
duplex without further carrier changes during a 30-second observation window.
Receive counters increased without input errors.

The module is loaded manually for this runtime test. It has not yet replaced
the in-tree driver through DKMS, so the change does not persist across reboot.

## Network validation

The adapter was connected to an Ubiquiti USW Pro 24 PoE SFP+ port configured
for 10 Gb/s full duplex. The switch and the driver agreed on link speed and
duplex, and the switch reported no port errors or dropped frames.

An isolated, temporary host address was used for the following tests:

- 1,000 regular ICMP packets: no loss
- 1,000 full-size 1,500-byte packets with DF set: no loss
- 50,000 small ICMP packets: no loss or TX-level warning
- sustained TCP transmit through a 1 Gb/s peer: about 939 Mb/s
- sustained TCP receive through a 1 Gb/s peer: about 771 Mb/s
- simultaneous TCP traffic: about 932 Mb/s TX and 748 Mb/s RX

The 1 Gb/s peer was the bottleneck in the TCP tests. A second 10 Gb/s host is
still required to measure full line-rate performance. During all tests the
driver reported no input, alignment, DFE, interrupt-full, PCI, watchdog, or
kernel errors.

## Known limitations and follow-up

- The driver exposes one RX/TX channel and one MSI interrupt. Actual 10 Gb/s
  scaling must be measured with a suitable peer.
- SFP module EEPROM and diagnostics are not exposed through `ethtool -m`.
- Pause-frame and EEE configuration are not exposed through ethtool.
- An extended `W=1` build succeeds but reports missing-prototype warnings in
  the legacy PHY glue and one unused local variable in the RX path.
- The previously proposed short-frame TX accounting change was not included:
  50,000 small packets did not reproduce the historical warning.
- On an unconfigured interface, IPv6 router advertisements may create a global
  address and default route when the link is brought up. Disable RA/autoconf or
  configure the interface explicitly before leaving it active on a production
  network.

## Validation checklist

- Build against the exact running Proxmox kernel headers.
- Keep the existing management NIC active during all tests.
- Confirm the detected PHY and MDIO address in the kernel log.
- Confirm module load/unload and interface up/down without warnings.
- Confirm link at 10 Gb/s with the intended SFP+ module or DAC.
- Test sustained bidirectional traffic and short frames.
- Reboot once and confirm that DKMS rebuilt and loaded the expected module.

Until these checks pass, this branch is experimental and should not replace a
working management interface.
