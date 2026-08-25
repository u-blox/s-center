# AT Scripting Quick Start Guide

Automate AT command sequences on your u-blox module using the **AT Scripting widget** in s-center. The widget supports a full script format with control flow, variables, wait conditions, and response parsing — plus a debugger with breakpoints and stepping.

## Hardware Requirements

- A u-blox module EVK (any supported model with Wi-Fi or Bluetooth depending on the script you want to run).
- A USB cable for connecting the EVK to your PC (included in the kit).
- For Wi-Fi / MQTT examples: a Wi-Fi access point with internet access.

## Prerequisites

- A u-blox module connected to your PC via USB.
- The product is already added in s-center (see the [Wi-Fi Quick Start](wifi-quick-start.md), Step 0, if you have not added the device yet).

---

## Section 1.  Widget Operation and Features

### Opening the Widget

1. Connect to your device from the left sidebar.
2. Click the **product menu** (three dots) next to your device name.
3. Select **AT Scripting**.

The AT Scripting widget opens on the canvas with an empty editor.

> **Important:** The AT Scripting widget does **not** display device I/O. All commands sent to the device and all responses received are visible in the **Terminal widget**. Open the Terminal widget side-by-side with the script editor for full visibility.

### Layout Overview

The widget is split into three areas:

| Area | Purpose |
|------|---------|
| **Toolbar** (top) | Run/debug controls, file operations, "Open Terminal" shortcut |
| **Editor** (center) | Multi-line script editor with a line-number / breakpoint gutter on the left |
| **Status bar** (bottom) | Current execution state, current line, and any error message |

### Editor Features

- **Multi-line editor** with monospaced font.
- **Line numbers** displayed in the left gutter.
- **Undo / Redo** with `Ctrl+Z` / `Ctrl+Y`.
- **Copy / Paste** with `Ctrl+C` / `Ctrl+V`.
- **Current-line marker** during execution, an arrow in the gutter shows which line is running, and the editor auto-scrolls to keep it visible.

### Breakpoints

- **Click** in the gutter (left margin) to toggle a breakpoint on a line.
- Press **F9** to toggle a breakpoint at the caret line.
- Breakpoints appear as red dots in the gutter.
- Execution **pauses before** the breakpointed line runs, allowing you to step through the rest manually.

### Run Controls

| Control | Shortcut | Description |
|---------|----------|-------------|
| **Run / Continue** | `F5` | Start execution, or continue from a paused state |
| **Pause** | `Space bar` | Pause execution at the next safe point |
| **Stop** | `Shift+F5` | Cancel execution immediately |
| **Restart** | `Ctrl+Shift+F5` | Reset to the start of the script (breakpoints are preserved) |
| **Step** | `F10` | Execute one line, then pause |

### Execution States

- **Running** — the script is actively executing.
- **Paused** — the script is stopped at a breakpoint or after stepping.
- **Stopped** — execution has finished, was cancelled, or has not started yet.

### File Operations

- **Open** — load a script from a `.txt`, `.at`, or `.script` file.
- **Save** (`Ctrl+S`) — save the current script to disk.

Scripts saved by s-center use a standard plain-text format — they can be edited externally and re-loaded.

### Open Terminal

The **Terminal** button in the toolbar opens (or brings to front) the Terminal widget for the same device. All AT traffic from the script is logged there in real time.

### Error Handling

| Error | Where it shows |
|-------|----------------|
| **Parse error** | Status bar (script does not start) |
| **`waitfor` timeout** | Status bar with the failing line number |
| **Device returned `ERROR`** | Status bar with the failing line number |
| **Device disconnected** | Status bar; execution stops |

The Terminal widget shows the full AT exchange leading up to any failure.

---

## Section 2 — AT Scripts

### 2.1 Syntax Reference

A script is a plain-text file with one statement per line.

#### Comments

```
# Full-line comment
AT+GMM      # Inline comment
```

#### AT Commands

Any line starting with `AT` (case-insensitive) is sent verbatim to the device. Execution waits for the device's `OK` / `ERROR` response before moving on.

```
AT
AT+GMM
AT+UBTLN="MyDevice"
```

AT commands are always sent **sequentially** — the script never

#### Labels and `goto`

A label is a line starting with `:` (or ending with `:` for the legacy form). Use `goto(label)` (or the legacy `goto label`) to jump.

```
:start
AT+GMM
goto(start)
```

#### Variables

Variables are untyped strings. Set with `set_variable(name, value)`. Reference with `{name}` (modern) or `%name` (legacy).

```
set_variable(counter, 0)
set_variable(counter, {counter} + 1)
println("Counter is now {counter}")
```

#### Wait Operations

| Statement | Units | Behavior |
|-----------|-------|----------|
| `sleep(<seconds>)` | seconds | Pause for N seconds |
| `wait(<ms>)` | milliseconds | Pause for N milliseconds |
| `waitfor(<pattern> [, <timeoutSec>])` | seconds (default 30) | Wait for a URC line matching `<pattern>`; fail on timeout |
| `waitforever()` | — | Wait indefinitely

```
waitfor(+STARTUP)         # default 30 s timeout
waitfor(+UEWSNU, 60)      # 60 s timeout
sleep(2)                  # 2 seconds
wait(500)                 # 500 ms
```

#### Response Extraction

`response(<pattern>, <var1> [, <var2> ...])` first looks for `<pattern>` in the **last AT command response**, then falls back to **waiting for a URC** (default 30 s) if no match was found. The text after the matched pattern is split by commas and assigned to the listed variables.

```
AT+USYLA?
response(+USYLA:, addr)               # "+USYLA:AABBCCDDEEFF" → addr = "AABBCCDDEEFF"

AT+USORS=0,512
response(+USORS:, socket_id, data)   # "+USORS:0,hello" → socket_id = "0", data = "hello"

response(+UEVT:, code, status)       # waits for a URC if no prior match
```

#### Output

```
println("Hello from the script")
echo("Same as println")
```

Both write to the script status / log area; they do **not** send anything to the device.

#### Conditionals

Two forms of `if` are supported:

```
# Inline if-with-goto: jumps to <label> if the condition is true
if({status} == 1, connected)
if({status} == 0, disconnected)
```

The legacy form is also accepted:

```
IF status == 1 GOTO connected
```

#### `for` Loops

```
for(i, 0, 5)
    println("Iteration {i}")
    AT+UBTGNS=0,14,{i}
:endfor_i
```

**must be named `endfor_<variable>`** — the parser

#### `while` Loops

```
set_variable(counter, 0)
while({counter} < 10)
    set_variable(counter, {counter} + 1)
    println("counter = {counter}")
endwhile()
```

`while` / `endwhile()` pairs are matched by nesting depth, so loops can be nested freely.


---

### 2.2 Examples

All examples below are complete, copy-pasteable scripts. Open the **Terminal widget** alongside the script editor to follow the AT exchange.

#### Example 1 — BLE Scan

Scans for nearby Bluetooth LE advertisers for 10 seconds and prints the results.

```
# --- BLE scan example ---
# Make sure the module is in Central mode and BLE is enabled.

AT
AT+UBTM=1           # Set BLE mode to Central
AT&W                # Store configuration
AT+CPWROFF          # Reboot to apply
waitfor(+STARTUP, 30)

# Start a 10-second active scan (type=0 all, mode=0 active, 10000 ms)
AT+UBTD=0,0,10000

# +UBTD: discoveries arrive as URCs while scanning
println("Scanning for 10 seconds...")
sleep(11)

println("Scan complete. See the Terminal widget for +UBTD: results.")
```

#### Example 2 — BLE GATT Client

Connects to a peripheral by Bluetooth address, discovers a service, reads a characteristic, and disconnects.

```
# --- BLE GATT client example ---
# Replace PEER_ADDR with the address of your peripheral (12 hex chars, public address).

set_variable(PEER_ADDR, AABBCCDDEEFF)

AT
AT+UBTM=1           # Central mode
AT&W                # Store configuration
AT+CPWROFF          # Reboot to apply
waitfor(+STARTUP, 30)

# Connect to the peer (public address)
AT+UBTC={PEER_ADDR}p
response(+UEBTC:, conn_handle, addr)
println("Connected, handle = {conn_handle}")

# Discover all primary services on the connection
AT+UBTGPSD={conn_handle}
sleep(2)

# Read characteristic at handle 14 (example — adjust for your peer)
AT+UBTGR={conn_handle},14
response(+UBTGR:, rd_handle, value)
println("Characteristic value: {value}")

# Disconnect
AT+UBTDC={conn_handle}
waitfor(+UEBTDC:, 10)
println("Disconnected.")
```

#### Example 3 — Wi-Fi Station Connection

Joins a Wi-Fi access point, waits for an IP address, and prints the assigned address.

```
# --- Wi-Fi Station example ---
# Edit SSID and PASSWORD to match your network.

set_variable(SSID, MyAccessPoint)
set_variable(PASSWORD, MySecretPassword)

AT
AT+UWSCP=0,"{SSID}"                # Set SSID
AT+UWSSW=0,"{PASSWORD}",0           # Set password (WPA2/WPA3)
AT+UWSC=0                           # Connect

# Wait for the network interface to come up
waitfor(+UEWSNU, 60)

# Read the assigned IPv4 address
AT+UWSNST=0
response(+UWSNST:, iface, ip)
println("Connected. IPv4 address: {ip}")
```

#### Example 4 — MQTT Publish / Subscribe

Connects to the public `broker.emqx.io` MQTT broker, subscribes to a topic, publishes a message, and waits for it to arrive back.

```
# --- MQTT publish/subscribe example ---
# Requires a Wi-Fi connection — run the Wi-Fi Station script first,
# or join the network manually before running this script.

set_variable(BROKER, broker.emqx.io)
set_variable(TOPIC, scenter/tutorial)
set_variable(MESSAGE, Hello from s-center scripting)

# Configure the MQTT client (host, port, client ID)
AT+UMQCP=0,"{BROKER}",1883,"scenter-script-001"

# Connect
AT+UMQC=0
waitfor(+UEMQC:, 30)

# Subscribe to the topic (QoS 0)
AT+UMQS=0,0,"{TOPIC}"
sleep(1)

# Publish to the same topic
AT+UMQPS=0,0,0,"{TOPIC}","{MESSAGE}"

# Wait for the inbound message notification and read it
waitfor(+UEMQD:, 30)
AT+UMQRS=0
response(+UMQRS:, mqtt_id, qos, topic, payload_len, payload)
println("Received on {topic}: {payload}")

# Disconnect
AT+UMQDC=0
println("Disconnected from broker.")
```

> **Note on AT command syntax:** The examples above use NORA-W36 AT commands. The exact parameter ordering may vary by firmware version. Always cross-check against the official [u-connectXpress AT command manual](https://github.com/u-blox/u-connectXpress) for your specific module.

---

## What's Next?

- Combine the examples above into longer end-to-end scripts (e.g. Wi-Fi ? MQTT ? publish telemetry in a loop).
- Use **breakpoints + Step (F10)** to walk through a script line-by-line while watching the Terminal widget.
- Save your scripts as `.script` files and share them with colleagues.
- Browse the [u-connectXpress AT command manual](https://github.com/u-blox/u-connectXpress) for the full command reference.
