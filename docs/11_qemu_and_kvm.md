# QEMU and KVM

[Chapter 3](03_type1_hypervisors.md) introduced KVM as a Linux kernel module that turns the host into a Type 1 hypervisor, and [Chapter 4](04_qemu.md) explained how QEMU pairs with KVM — KVM handles CPU and memory virtualization via hardware extensions while QEMU provides the device model. This chapter covers the practical side: enabling KVM acceleration, verifying hardware support, and handling nested virtualization.

## Enabling KVM Acceleration

Without KVM, QEMU translates every guest CPU instruction through its software emulation engine (TCG). With KVM enabled, guest code runs directly on the host CPU — the performance difference is dramatic.

The original way to enable KVM is the `-enable-kvm` flag:

```bash
qemu-system-x86_64 -m 512 -hda alpine_x86.qcow2 -boot c -enable-kvm
```

QEMU also provides a more flexible `-accel` option, which supports specifying multiple accelerators:

```bash
qemu-system-x86_64 -m 512 -hda alpine_x86.qcow2 -boot c -accel kvm
```

Both commands are equivalent. The `-accel` syntax is the modern, preferred form.

## Available Accelerators

To list the accelerators compiled into your QEMU binary:

```bash
qemu-system-x86_64 -accel help
```

On a Linux host, the output typically includes:

```
Accelerators supported in QEMU binary:
tcg
kvm
```

| Accelerator | Description                                                                 |
|-------------|-----------------------------------------------------------------------------|
| **tcg**     | QEMU's built-in software emulator (Tiny Code Generator) — always available  |
| **kvm**     | Linux KVM — near-native speed via hardware virtualization extensions        |

On Windows, a different set of accelerators may appear:

```
Accelerators supported in QEMU binary:
tcg
whpx
```

**WHPX** (Windows Hypervisor Platform) is a set of Windows APIs that allows third-party applications like QEMU to use the host CPU's hardware-assisted virtualization features. It serves the same role on Windows that KVM serves on Linux — enabling hardware-accelerated VM execution.

## Checking KVM Support

Before enabling KVM, verify that your system supports it. Install the `cpu-checker` package:

```bash
sudo apt install cpu-checker
```

Then run the `kvm-ok` utility:

```bash
sudo kvm-ok
```

If KVM is **not** available, the output will be:

```
INFO: Your CPU does not support KVM extensions
KVM acceleration can NOT be used
```

If KVM **is** available:

```
INFO: /dev/kvm exists
KVM acceleration can be used
```

Common reasons KVM may be unavailable:

- **Hardware extensions disabled** — VT-x or AMD-V may be turned off in the BIOS/UEFI settings
- **Running inside a VM** — The outer hypervisor may not expose virtualization extensions to the guest (see the next section)
- **Another hypervisor is active** — On some systems, only one hypervisor can use the hardware extensions at a time

## Nested Virtualization

If you are running QEMU inside a virtual machine (e.g., an Ubuntu VM on VirtualBox or VMware), KVM will not work by default. The outer hypervisor does not expose hardware virtualization extensions to the guest unless **nested virtualization** is explicitly enabled.

Nested virtualization allows a guest VM to act as a hypervisor and run its own VMs. This is achieved by having the outer hypervisor pass through the CPU's virtualization extensions (Intel VT-x / AMD-V) to the guest.

### VirtualBox

In VirtualBox, enable nested virtualization through:

**Settings → System → Processor → Enable Nested VT-x/AMD-V**

If the checkbox is grayed out, enable it from the command line:

```bash
VBoxManage modifyvm <VMName> --nested-hw-virt on
```

### VMware

In VMware Workstation or Fusion, the setting is:

**Virtual Machine Settings → Processors → Virtualize Intel VT-x/EPT or AMD-V/RVI**

### Verification

After enabling nested virtualization in the outer hypervisor, boot the guest and run `kvm-ok` again:

```bash
sudo kvm-ok
```

```
INFO: /dev/kvm exists
KVM acceleration can be used
```

You can now use `-accel kvm` or `-enable-kvm` in your QEMU commands inside the guest VM.

---

[← Previous: Direct Linux Boot](10_direct_linux_boot.md) | [Next: User-mode Emulation →](12_qemu_user_emulation.md)
