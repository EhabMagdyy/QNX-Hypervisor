# QNX Hypervisor on Raspberry Pi 4b
> <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/6d3cf74e-6ac9-4fa1-a589-0cc2fc462604" />

## Overview

This guide explains how to configure and run the **QNX Hypervisor** on a **Raspberry Pi 4b**, including:

* Configuring a QNX guest
* Building the Hypervisor disk image
* Deploying the image to an SD card
* Booting the Hypervisor
* Launching guests
* Shutting down guests and host safely

---

## Prerequisites

#### Before proceeding:
- ensure that the `Raspberry Pi 4 Board Support Package (BSP)` is installed in your QNX Software Development Platform (SDP) environment.
- also, you may need `QNX hypervisor arm guests` from the SW Center as well, if you need modify the guest image.

---

## Directory Structure

```text
images/
├── bcm2711-rpi-4-b.dtb
├── overlays/
├── config.txt
├── start4.elf
├── definitions.m4
├── disk.cfg
├── fixup4.dat
├── guests/
|    └──qnx-guest
|        ├── qnx800-guest-1.ifs
|        └── qnx800-guest-1.qvmconf
├── part_dos_boot.build
├── part_qnx_data.build
├── rpi4-hypervisor.build
└── Makefile
```

---

## Configure QNX Guest

Navigate to your Raspberry Pi 4 BSP directory.

Create the guest directory:

```bash
mkdir -p images/guests/qnx-guest
```

Copy the following files from QNX Hypervisor Package downloaded from SW Center `images/guest-1/` into `images/guests/qnx-guest/`:

| File                     | Description             |
| ------------------------ | ----------------------- |
| `qnx800-guest-1.ifs`     | QNX guest IFS image     |
| `qnx800-guest-1.qvmconf` | QNX guest configuration |

Example:

```text
images/
└── guests/
    └── qnx-guest/
        ├── qnx800-guest-1.ifs
        └── qnx800-guest-1.qvmconf
```

---

## Prepare Hypervisor Disk Image

### Create Disk Partitions

The Hypervisor disk image requires:

| Partition             | Purpose         |
| --------------------- | --------------- |
| DOS Boot Partition    | Hypervisor Host |
| QNX6FS Data Partition | Guest Images    |

Place the following files inside the `images/` directory:

```text
disk.cfg
part_dos_boot.build
part_qnx_data.build
```

---

## Transfer Firmware Files

Ensure the following Raspberry Pi firmware files are present (Boot Partition files):

```text
bcm2711-rpi-4-b.dtb
overlays/
start4.elf
fixup4.dat
```

---

## Build Disk Image

Modified the `Makefile` inside the `images/` directory to build:

* Hypervisor IFS
* Hypervisor Disk Image

From the BSP root directory:

```bash
make
```

After the build completes:

```text
images/disk.img
```

will be generated.

---

## Transfer Disk Image to SD Card

```bash
sudo dd if=disk.img of=/dev/sdX bs=4M status=progress

# Example output:
# 5968494592 bytes (6.0 GB, 5.6 GiB) copied, 411 s, 14.5 MB/s
# 1425+0 records in
# 1425+0 records out
# 5976883200 bytes (6.0 GB, 5.6 GiB) copied, 411.763 s, 14.5 MB/s
```

Flush remaining writes:

```bash
sync
```

---

## Booting QNX Hypervisor

### Hardware Connections

Connect:

* SD card containing the Hypervisor image
* USB-TTL between your PC & RPi
* Ethernet cable
* Power supply

---

### Serial Console

Open a serial terminal:

```bash
sudo picocom /dev/ttyUSB0 -b 115200
```

---

### Expected Boot Logs

```text
Enabling EL1 host hypervisor support
MMU: 16-bit ASID 44-bit PA TCR_EL1=00000014b5183519
GICv2: 256 interrupts
GICv2: routing SPIs to gic cpu 0
cpu0: MPIDR=0000000080000000
cpu0: MIDR=410fd083 Cortex-A72 r0p3
cpu0: DCZID_EL0=0x00000004 (64 bytes)
cpu0: CWG=4 ERG=4 Dminline=4 Iminline=4 PIPT
cpu0: CLIDR=a200023 LoUU=1 LoC=2 LoUIS=1
cpu0: L1 Icache 48K linesz=64 set/way=256/3
cpu0: L1 Dcache 32K linesz=64 set/way=256/2
cpu0: L2 Unified 1024K linesz=64 set/way=1024/16
cpu0: GICv2 cpu interface 0
Loading IFS...done
cpu1: MPIDR=0000000080000001
cpu1: MIDR=410fd083 Cortex-A72 r0p3
cpu1: DCZID_EL0=0x00000004 (64 bytes)
cpu1: CWG=4 ERG=4 Dminline=4 Iminline=4 PIPT
cpu1: CLIDR=a200023 LoUU=1 LoC=2 LoUIS=1
cpu1: L1 Icache 48K linesz=64 set/way=256/3
cpu1: L1 Dcache 32K linesz=64 set/way=256/2
cpu1: L2 Unified 1024K linesz=64 set/way=1024/16
cpu1: GICv2 cpu interface 1
cpu2: MPIDR=0000000080000002
cpu2: MIDR=410fd083 Cortex-A72 r0p3
cpu2: DCZID_EL0=0x00000004 (64 bytes)
cpu2: CWG=4 ERG=4 Dminline=4 Iminline=4 PIPT
cpu2: CLIDR=a200023 LoUU=1 LoC=2 LoUIS=1
cpu2: L1 Icache 48K linesz=64 set/way=256/3
cpu2: L1 Dcache 32K linesz=64 set/way=256/2
cpu2: L2 Unified 1024K linesz=64 set/way=1024/16
cpu2: GICv2 cpu interface 2
cpu3: MPIDR=0000000080000003
cpu3: MIDR=410fd083 Cortex-A72 r0p3
cpu3: DCZID_EL0=0x00000004 (64 bytes)
cpu3: CWG=4 ERG=4 Dminline=4 Iminline=4 PIPT
cpu3: CLIDR=a200023 LoUU=1 LoC=2 LoUIS=1
cpu3: L1 Icache 48K linesz=64 set/way=256/3
cpu3: L1 Dcache 32K linesz=64 set/way=256/2
cpu3: L2 Unified 1024K linesz=64 set/way=1024/16
cpu3: GICv2 cpu interface 3

System page at phys:0000000000016000 user:ffffff8040208000 kern:ffffff8040204000
Starting next program at vffffff8060074950
syspage::hypinfo::flags=0x00000001
No #1262888 WA needed for CPU0
No #1262888 WA needed for CPU2
No #1262888 WA needed for CPU1
No #1262888 WA needed for CPU3
 
Welcome to QNX 8.0.0 on RaspberryPi4B !
 
Starting wdtkick ...
Starting I2C driver ...
Starting PCI Server ...
Starting serial driver (/dev/ser1)
Starting SPI master driver ...
Starting SDMMC driver (/dev/sd0)
Path=0 - bcm2711
 target=0 lun=0     Direct-Access(0) - SDMMC: SD16G Rev: 2.0
Inform vc to load vl805 firmware
Starting USB xHCI controller in the host mode (/dev/usb/*)...
Mounting /dev/sd0t179 on / ...
Starting networking ...
Creating example ram disks for guests to use
Path=0 - 
 target=0 lun=0     Direct-Access(0) - ram  Rev: 
net.link.bridge.inherit_mac: 0 -> 1
Starting DHCP client ...
Starting SSH daemon ...
Starting devc-pty manager ...
Starting qconn daemon ...
Starting board customize script ...
Running user's startup script ...
Starting shell ...
# 
```

---

## Assign an IP address to the Ethernet

```sh
ifconfig bridge0 192.168.50.100 netmask 255.255.255.0
route add default 192.168.50.1
```

---

## From another terminal you can ssh to your host

```sh
ssh root@192.168.50.100
# password: root
```

---

## Starting a QNX Guest

Navigate to the guest directory:

```bash
cd /guests/qnx-guest
```

Launch the guest:

```bash
qvm @qnx800-guest-1.qvmconf
```

---

### Expected Guest Output

```text
[qnx800-guest-1.qvmconf:58] Vendor feature VIRTIO_NET_VF_GUEST_TS configured at bit 127
[qnx800-guest-1.qvmconf:70] Vendor feature VIRTIO_NET_VF_GUEST_TS configured at bit 127
GICv3: using memory-mapped interface, full software emulation not supported
PE does not support PAUTH feature
gic_v3_initialize(), setting 'use_memory_mapped_callouts'
syspage::hypinfo::flags=0x00000000
 
Welcome to QNX 8.0.0 on ARMv8_Foundation_Model !
 
Starting devf-ram filesystem ...
Starting networking ...
Starting DHCP client ...
Starting SSH daemon ...
Starting devc-pty manager ...
Starting qconn daemon ...
Virtual networking comes up by default if virtio-net devices are configured for this guest

A virtual block device can be started by running /scripts/block-start.sh
   (this requires vdev-virtio-blk is configured for this guest)
   (Note that it assumes you are using a blank RAM disk as host device)

Virtual shared memory device demo driver can be started by running /scripts/shmem-start.sh
   (this requires vdev-shmem is configured in this guest)

Virtual watchdog device can be started and stopped by running /scripts/watchdog-start.sh and watchdog-stop.sh
   (this requires a vdev-wdt-* is configured in this guest)

Note: the scripts above assume that you have located your virtual devices at particular loc/intr values
      that match the values passed to the corresponding driver

Starting shell ...
[armv8 guest QNX 8.x guest 1]% 
```

---

## Stopping Guests

To terminate running guests from the QNX Host:

```bash
# to kill all guests
slay qvm
# to kill a specific guest
pidin arg | grep qvm    # get your guest's pid
kill -9 <qvm-pid>       # kill it
```

Alternatively, shut down the guest from inside the guest OS.

```bash
shutdown
# Guest qnx-guest-1 has been shut down
```

---

## Shutting Down QNX Host

Safely power off the Raspberry Pi 4 host:

```bash
shutdown
```

---

## Summary

The deployment flow is:

```text
Build Guest Images
        │
        ▼
Configure Guest Directories
        │
        ▼
Build Hypervisor Disk Image
        │
        ▼
Flash SD Card
        │
        ▼
Boot QNX Hypervisor Host
        │
        └── Start QNX Guest
```

This setup enables simultaneous execution of multiple guest operating systems under the QNX Hypervisor on Raspberry Pi 4b.
