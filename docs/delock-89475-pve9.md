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

1. Linux 7.x / Proxmox VE 9 build compatibility, if required.
2. Short-frame TX accounting, after isolated review and testing.
3. DKMS packaging and installation documentation.

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
