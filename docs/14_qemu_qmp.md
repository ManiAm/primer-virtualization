# QEMU Machine Protocol (QMP)

The [previous chapter](13_qemu_monitor.md) covered the QEMU Monitor — a human-readable interface for interactive VM management. QMP (QEMU Machine Protocol) serves the same purpose but is designed for **programmatic control**. It uses JSON-based messages exchanged over a socket (Unix domain socket or TCP), making it suitable for automation, scripting, and integration with management tools like libvirt.

## Starting the QMP Server

To enable QMP, add the `-qmp` option when launching QEMU. This creates a Unix domain socket that QMP clients can connect to:

```bash
qemu-system-x86_64 \
    -m 512 \
    -hda alpine_x86.qcow2 \
    -boot c \
    -nographic \
    -qmp unix:./qmp-sock,server,nowait
```

The `server,nowait` options tell QEMU to listen for connections without blocking VM startup.

## QMP Handshake

Every QMP session follows a three-step handshake before commands can be issued:

1. **Greeting** — Upon connection, the server sends a JSON greeting that includes the QEMU version and supported capabilities:

```json
{"QMP": {"version": {"qemu": {"micro": 4, "minor": 0, "major": 8}}, "capabilities": ["oob"]}}
```

2. **Capability negotiation** — The client must send the `qmp_capabilities` command to acknowledge the greeting and enter command mode:

```json
{"execute": "qmp_capabilities"}
```

The server responds with:

```json
{"return": {}}
```

3. **Command mode** — The session is now ready to accept commands.

## Interacting via Telnet

For manual exploration, you can interact with QMP using telnet. Since telnet cannot connect to Unix domain sockets directly, use `socat` to bridge the socket to a TCP port:

```bash
socat TCP-LISTEN:1234,reuseaddr,fork UNIX-CONNECT:./qmp-sock &
```

Then connect with telnet:

```bash
telnet localhost 1234
```

After the greeting appears, complete the handshake and issue commands:

```
{"execute": "qmp_capabilities"}
{"execute": "query-status"}
{"execute": "query-commands"}
```

## QMP Shell

Typing raw JSON is tedious. QEMU includes **qmp-shell**, a user-friendly command-line tool that handles the handshake automatically and provides tab completion for commands.

The script is included in the QEMU source tree. Clone the repository and navigate to the scripts directory:

```bash
git clone https://github.com/qemu/qemu.git
cd qemu/scripts/qmp
```

Connect to a running QEMU instance:

```bash
./qmp-shell ./qmp-sock
```

```
Welcome to the QMP low-level shell!
Connected to QEMU 8.0.4

(QEMU)
```

Use tab completion to discover available commands:

```
(QEMU) query-<TAB>
```

Example — check whether KVM is enabled:

```
(QEMU) query-kvm
{"return": {"enabled": false, "present": true}}
```

## QMP Client in Python

For automation, you can write a QMP client in any language that supports Unix sockets and JSON. The following Python example demonstrates the connection handshake and command execution:

```python
import json
import socket


class QMPClient:

    def __init__(self, path):
        self.sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        self.sock.connect(path)

        greeting = self._receive()
        print("Connected to QMP server:", greeting)

        result = self._negotiate_capabilities()
        print("Capability negotiation:", result)

    def _receive(self):
        response = b""
        while True:
            chunk = self.sock.recv(4096)
            response += chunk
            if b"\n" in chunk:
                break
        return json.loads(response.decode("utf-8"))

    def _send(self, cmd):
        self.sock.sendall(json.dumps(cmd).encode("utf-8") + b"\n")

    def _negotiate_capabilities(self):
        self._send({"execute": "qmp_capabilities"})
        return self._receive()

    def execute(self, command, arguments=None):
        cmd = {"execute": command}
        if arguments:
            cmd["arguments"] = arguments
        self._send(cmd)
        return self._receive()

    def close(self):
        self.sock.close()


if __name__ == "__main__":
    client = QMPClient("./qmp-sock")

    print("QEMU version:", client.execute("query-version"))
    print("QEMU status:", client.execute("query-status"))
    print("Character devices:", client.execute("query-chardev"))

    client.close()
```

The script connects to the QMP socket, performs the capability negotiation, and then issues three queries: the QEMU version, VM running status, and attached character devices.

For the complete list of QMP commands and their parameters, see the [QMP reference manual](https://qemu-project.gitlab.io/qemu/interop/qemu-qmp-ref.html).

---

[← Previous: QEMU Monitor](13_qemu_monitor.md) | [Next: Host-Guest File Sharing →](15_host_guest_file_sharing.md)
