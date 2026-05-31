# QNX Hypervisor Architecture on Raspberry Pi 4

## Overview

The QNX Hypervisor 8.0 provides a secure virtualization platform for running multiple operating systems simultaneously on ARM-based hardware. On the Raspberry Pi 4 (Broadcom BCM2711), the hypervisor architecture differs from traditional ARM Type-1 hypervisors due to hardware-specific limitations surrounding Virtualization Host Extensions (VHE).

Understanding how Exception Levels (ELs), Stage-2 memory translation, trap routing, and Virtual Machine (VM) emulation interact is essential when building:

* Secure edge gateways
* Automotive systems
* Industrial controllers
* Mixed-criticality embedded systems
* Isolated Over-The-Air (OTA) update infrastructures
* Safety-certified embedded platforms

This document explains how the QNX Hypervisor uses the ARMv8-A privilege model to provide strong isolation while remaining compatible with Raspberry Pi 4 hardware.

---

# ARMv8-A Privilege Architecture

ARMv8-A processors execute software at different privilege levels known as **Exception Levels (ELs)**.

Each level provides a different degree of control over the hardware.

```text
+--------------------------------------+
| EL3 - Secure Monitor                 |
| TrustZone, Secure Boot, Firmware     |
+--------------------------------------+
| EL2 - Hypervisor                     |
| Virtualization Control               |
+--------------------------------------+
| EL1 - Operating System Kernels       |
| Linux, QNX, Android                  |
+--------------------------------------+
| EL0 - User Applications              |
| Shells, Browsers, Processes          |
+--------------------------------------+
```

Whenever software attempts an operation that requires higher privileges, the processor generates an exception and transitions execution to a higher Exception Level.

---

# EL0 - User Space

EL0 is the least privileged execution level.

Applications execute here and cannot directly access:

* Physical memory
* Device registers
* MMU configuration
* Interrupt controller registers
* Hypervisor resources

Examples include:

```text
bash
ssh
qvm
web browsers
user applications
```

Every hardware access request must pass through the operating system kernel.

---

# EL1 - Operating System Kernel

EL1 is where a traditional operating system kernel executes.

Responsibilities include:

* Process scheduling
* Memory management
* Interrupt handling
* System call processing
* Device driver management

Examples:

* Linux kernel
* QNX Neutrino microkernel (procnto)
* Android kernel

Without virtualization support, EL1 has complete ownership of the hardware.

---

# EL2 - Hypervisor

EL2 exists specifically for virtualization.

The processor routes virtualization-related events to EL2, including:

* Accesses to virtual devices
* Stage-2 MMU faults
* Guest interrupt injections
* Virtual CPU management
* Guest context switching

EL2 is responsible for maintaining isolation between virtual machines.

```text
Guest Linux
     |
Guest QNX
     |
Guest Android
     |
     V
+----------------+
| Hypervisor EL2 |
+----------------+
       |
       V
 Hardware
```

A guest believes it owns the hardware while EL2 secretly controls access to all resources.

---

# EL3 - Secure Monitor

EL3 is the highest privilege level.

It is normally occupied by:

* ARM Trusted Firmware (ATF)
* Secure Boot firmware
* TrustZone monitor

Responsibilities include:

* Secure world transitions
* Key management
* Cryptographic services
* Secure boot validation

The QNX Hypervisor generally operates below EL3.

---

# Memory Virtualization

Virtualization requires more than CPU isolation.

The hypervisor must also prevent guests from accessing each other's memory.

ARM accomplishes this through two levels of address translation.

## Stage 1 Translation

Managed by the guest OS.

```text
Virtual Address
       |
       V
Guest Physical Address
```

The guest kernel believes it controls physical memory.

---

## Stage 2 Translation

Managed by the hypervisor.

```text
Guest Physical Address
          |
          V
Real Physical Address
```

Combined view:

```text
Application VA
      |
      V
Guest Physical Address
      |
      V
Real Physical Memory
```

This mechanism ensures:

* Memory isolation
* Secure guest separation
* Controlled DMA access
* Prevention of guest memory corruption

Stage-2 translation hardware always belongs to EL2.

This is the key reason EL2 must always exist, even when the host kernel runs at EL1.

---

# Virtualization Host Extensions (VHE)

## Traditional ARM Virtualization

Without VHE:

```text
Host Kernel ---> EL1
Hypervisor ---> EL2
Guests -------> EL1
```

Every time the host performs virtualization operations, execution switches between EL1 and EL2.

This introduces:

* Additional latency
* More context switches
* Greater software complexity

---

## VHE Architecture

Virtualization Host Extensions simplify the design.

With VHE:

```text
Host Kernel ---> EL2
Guests -------> EL1
```

The host kernel effectively becomes part of the hypervisor.

Benefits include:

* Fewer privilege transitions
* Reduced trap overhead
* Faster guest scheduling
* Better overall performance

Modern ARM server processors commonly use VHE.

---

# Raspberry Pi 4 VHE Limitation

The Raspberry Pi 4 uses the Broadcom BCM2711 SoC with Cortex-A72 cores.

Although the Cortex-A72 architecture supports virtualization extensions and VHE, practical limitations exist in the platform implementation.

Certain subsystems do not behave reliably under a full EL2-host virtualization model.

As a result, QNX does not support running the complete host operating system at EL2 on Raspberry Pi 4.

Instead, QNX requires:

```bash
-Q enable,el1-host
```

during startup image creation.

This enables the EL1-host architecture.

---

# QNX EL1-Host Architecture

The EL1-host design separates responsibilities across three privilege levels.

```text
                +----------------+
                |      EL2       |
                | Trap Handler   |
                +----------------+
                        |
                        V
                +----------------+
                |      EL1       |
                | QNX Host OS    |
                +----------------+
                        |
                        V
                +----------------+
                |      EL0       |
                | qvm Process    |
                +----------------+
```

---

# What Runs at EL2?

Contrary to many hypervisors, almost nothing runs at EL2.

EL2 contains:

* Exception vectors
* Minimal trap handlers
* Context save/restore code
* Stage-2 MMU management

EL2 does NOT contain:

* Device emulators
* Scheduling logic
* Drivers
* Filesystems
* Networking

Think of EL2 as a traffic controller.

Its only responsibility is catching virtualization exceptions and forwarding them.

---

# What Runs at EL1?

EL1 hosts:

## QNX Host Kernel

The host system executes:

```text
procnto
networking stack
filesystem
drivers
resource managers
```

It behaves as a normal QNX operating system.

---

## Guest Kernels

Guest operating systems also execute at EL1:

```text
Linux Guest
QNX Guest
Other ARM Guests
```

The guest kernel believes it has full control of the machine.

However, Stage-2 translation prevents access to unauthorized resources.

---

# What Runs at EL0?

The most interesting component is:

```text
qvm
```

The QNX Virtual Machine Manager.

Unlike monolithic hypervisors, QNX places device emulation in user space.

Examples include:

* Virtual UARTs
* VirtIO devices
* Virtual storage
* Virtual networking
* Shared memory devices

Because qvm runs at EL0:

* A bug cannot directly crash the host kernel
* Device emulators are isolated
* Security is improved
* Fault recovery is easier

This follows the same microkernel philosophy used throughout QNX.

---

# Trap Flow Example

Suppose a Linux guest accesses a virtual UART.

## Step 1

Guest executes:

```text
UART Write
```

---

## Step 2

Hardware detects the access requires virtualization handling.

```text
Guest EL1
     |
     V
Trap
```

---

## Step 3

CPU enters EL2.

```text
EL2 Stub
```

The stub:

* Saves CPU context
* Records trap information
* Redirects handling

---

## Step 4

Control returns to the host.

```text
EL1 Host
```

The host determines that emulation is required.

---

## Step 5

Request is delivered to qvm.

```text
EL0 qvm
```

qvm emulates the UART behavior.

---

## Step 6

Result returns back through the chain.

```text
qvm
  |
Host EL1
  |
EL2
  |
Guest EL1
```

Execution resumes as if real hardware existed.

---

# Why This Architecture Is Valuable

The EL1-host architecture provides several benefits.

## Stability

Avoids Raspberry Pi 4 VHE limitations.

## Security

Device emulation runs in isolated user-space processes.

## Reliability

Faults inside virtual devices do not compromise EL2.

## Microkernel Alignment

Consistent with QNX design philosophy.

## Mixed-Criticality Support

Allows safety-critical and non-critical systems to coexist.

Examples:

```text
Guest 1:
Safety Controller

Guest 2:
Linux OTA Update System

Guest 3:
Remote Diagnostics System
```

A failure in one guest remains contained.

---

# Architectural Summary

The complete execution flow is:

```text
 Guest OS (EL1)
       |
       V
 Hardware Trap
       |
       V
    EL2 Stub
       |
       V
Host Kernel (EL1)
       |
       V
qvm Emulator (EL0)
       |
       V
Host Kernel (EL1)
       |
       V
      EL2
       |
       V
 Guest OS (EL1)
```

Rather than placing the entire host operating system at EL2 using VHE, QNX uses a lightweight EL2 trap layer, a fully featured host OS at EL1, and user-space device emulation at EL0. This design avoids Raspberry Pi 4 virtualization limitations while preserving strong isolation, security, and reliability characteristics expected from a modern embedded hypervisor.
