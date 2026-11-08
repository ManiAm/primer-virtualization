# QEMU Monitor

The QEMU Monitor is an interactive command-line interface built into QEMU for inspecting and controlling a running virtual machine. Through the monitor, you can query VM state, manage attached devices, take snapshots, pause or resume execution, and more — all without shutting down the guest.

This is a **human-readable** interface designed for interactive use. The next chapter covers [QMP](14_qemu_qmp.md), a JSON-based protocol intended for programmatic control.

## Accessing the Monitor (Graphical Mode)

When QEMU runs with a graphical window, the monitor is available as a second console tab within the QEMU window.

Start a VM:

```bash
qemu-system-x86_64 \
    -m 512 \
    -hda alpine_x86.qcow2 \
    -boot c
```

Once the QEMU window appears, press `Ctrl+Alt+2` to switch to the monitor console. The prompt changes to `(qemu)`. Press `Ctrl+Alt+1` to switch back to the guest display.

## Accessing the Monitor (nographic Mode)

Previous chapters in this primer use `-nographic`, which disables the graphical window and routes the guest console to the terminal. In this mode, the terminal is **shared** between the guest and the QEMU monitor. You switch between them using the `Ctrl+A` escape sequence:

| Key Sequence  | Action                                           |
|---------------|--------------------------------------------------|
| `Ctrl+A, C`   | Toggle between the guest console and the monitor |
| `Ctrl+A, H`   | Print help for all escape sequences              |
| `Ctrl+A, X`   | Terminate the QEMU process                       |

To try it, start a VM with `-nographic`:

```bash
qemu-system-x86_64 \
    -m 512 \
    -hda alpine_x86.qcow2 \
    -boot c \
    -nographic
```

Press `Ctrl+A` then `C` (two separate keystrokes) to enter the monitor. The `(qemu)` prompt appears. Press the same sequence again to return to the guest.

### Alternative: Telnet Access

If you prefer the monitor in a **separate terminal window**, redirect it to a Telnet port:

```bash
qemu-system-x86_64 \
    -m 512 \
    -hda alpine_x86.qcow2 \
    -boot c \
    -nographic \
    -monitor telnet:127.0.0.1:1234,server,nowait
```

Then connect from another terminal:

```bash
telnet 127.0.0.1 1234
```

The `server,nowait` options tell QEMU to listen for connections but not block VM startup waiting for a client.

## Common Monitor Commands

The monitor provides a wide range of commands. Some frequently used ones:

| Command              | Description                                      |
|----------------------|--------------------------------------------------|
| `help`               | List all available commands                      |
| `info block`         | Show attached block devices (disks, CD-ROMs)     |
| `info network`       | Show network interfaces                          |
| `info snapshots`     | List VM snapshots                                |
| `info status`        | Show VM running state (running, paused, etc.)    |
| `savevm <name>`      | Take a named snapshot of the VM state            |
| `loadvm <name>`      | Restore a previously saved snapshot              |
| `stop`               | Pause VM execution                               |
| `cont`               | Resume VM execution                              |
| `system_reset`       | Reset the guest (like pressing the reset button) |
| `quit`               | Terminate the QEMU process                       |

### Example: Inspecting Block Devices

The `info block` command shows all block devices attached to the VM:

```
(qemu) info block
ide0-hd0 (#block170): alpine_x86.qcow2 (qcow2)
    Attached to:      /machine/unattached/device[23]
    Cache mode:       writeback

floppy0: [not inserted]
    Attached to:      /machine/unattached/device[16]
    Removable device: not locked, tray closed

sd0: [not inserted]
    Removable device: not locked, tray closed
```

This shows that `alpine_x86.qcow2` is attached as the primary hard disk in QCOW2 format, while the floppy and SD card slots are empty.

For the complete list of monitor commands, see the [QEMU Monitor documentation](https://qemu-project.gitlab.io/qemu/system/monitor.html).

---

[← Previous: User-mode Emulation](12_qemu_user_emulation.md) | [Next: QEMU Machine Protocol (QMP) →](14_qemu_qmp.md)
