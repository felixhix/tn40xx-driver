introduction
============
Dynamic Kernel Module Support (DKMS) builds Linux kernel modules whose sources reside outside the kernel source tree. It automates rebuilding of such modules when a new kernel is installed.

The `delock-89475-pve9` branch uses the DKMS version
`delock-89475-pve9-1` and builds only the TI/TLK10232 PHY support required
by the Delock 89475. The module is rebuilt automatically after kernel updates.

install
=======
    git clone -b delock-89475-pve9 https://github.com/felixhix/tn40xx-driver.git /usr/src/tn40xx-delock-89475-pve9-1
    dkms add -m tn40xx -v delock-89475-pve9-1

build driver for current kernel
===============================
    dkms install -m tn40xx -v delock-89475-pve9-1

build driver for specific kernel version
========================================
    dkms install -m tn40xx -v delock-89475-pve9-1 -k [kernel_version]

uninstall
=========
This will remove module for all kernel versions

    dkms remove -m tn40xx -v delock-89475-pve9-1 --all
