# PSU-EXT Software Quick Start

The fastest way to run PSU-EXT is to install the latest published release. The
installer downloads checksum-verified release files, keeps the application and
its data under your user profile, and configures the Proxy, Script Runner, and
frontend as user-level services without requiring administrator privileges.

## Install the Latest Release

Use the bootstrap command for your platform to download the `psu-ext` command.

### Linux x86-64

```sh
curl -fsSL https://raw.githubusercontent.com/PSU-Ext-Org/psu-ext-software/main/psu-install/install-linux.sh | sh
export PATH="$HOME/.local/bin:$PATH"
```

### Apple Silicon macOS

```sh
curl -fsSL https://raw.githubusercontent.com/PSU-Ext-Org/psu-ext-software/main/psu-install/install-macos.sh | sh
export PATH="$HOME/.local/bin:$PATH"
```

### Windows 11 x64

```powershell
Invoke-WebRequest `
  https://raw.githubusercontent.com/PSU-Ext-Org/psu-ext-software/main/psu-install/install-windows.ps1 `
  -OutFile install-windows.ps1
pwsh -NoProfile -File .\install-windows.ps1
$env:Path += ";$env:LOCALAPPDATA\Programs\PSU-EXT"
```

The initial macOS and Windows binaries are unsigned, so the operating system
may ask you to approve them the first time they run.

After installing the command, download the current PSU-EXT release and start
all services:

```sh
psu-ext install
psu-ext start
```

`psu-ext start` prints the local URL to open in your browser. For supported
platforms, installation locations, updates, service commands, and
troubleshooting, see the
[`psu-install` README](https://github.com/PSU-Ext-Org/psu-ext-software/blob/main/psu-install/README.md).

## Manual Installation From Sources

This guide starts the PSU-EXT WebSocket Proxy, Script Runner, and browser
frontend from a local checkout of the `psu-ext-software` repository. The three
applications run as separate processes:

| Application | Default address | Purpose |
| --- | --- | --- |
| Proxy | `ws://localhost:8080/ws/scpi` | Connects the browser dashboard to TCP or USB SCPI devices |
| Frontend | `http://localhost:5173` | Provides the dashboard, configuration UI, and script IDE |
| Script Runner | `http://localhost:8081` | Enables IDE script management and execution against configured devices |

### Prerequisites

Install the following software before starting:

- Java 17 or newer. Confirm with `java -version`.
- Node.js 20 or newer and npm. Confirm with `node --version` and
  `npm --version`.
- Git, if the source repository has not already been downloaded.
- Permission to access the required serial port when using a USB device.

The backend includes Maven Wrapper scripts, so a separate Maven installation is
not required. The first Maven or npm run may need internet access to download
dependencies.

The commands below assume this source layout:

```text
psu-ext-software/
├── psu-be/                 # Java backend workspace
│   ├── psu-be-proxy/
│   └── psu-be-script-runner/
└── psu-fe/                 # React frontend
```

Run each backend command from `psu-ext-software/psu-be`. Relative paths such as
`./devices.json`, `./script-definitions`, and `./script-storage` are resolved
from that working directory.

### 1. Start the Proxy

Open a terminal in `psu-ext-software/psu-be` and start the Proxy.

PowerShell:

```powershell
cd psu-ext-software\psu-be
.\mvnw.cmd -pl psu-be-proxy -am spring-boot:run
```

Bash:

```bash
cd psu-ext-software/psu-be
./mvnw -pl psu-be-proxy -am spring-boot:run
```

Leave this terminal running. Unless overridden, Spring Boot listens on port
`8080` and exposes the WebSocket endpoint at `/ws/scpi`.

### 2. Install and start the frontend

Open a second terminal in `psu-ext-software/psu-fe`.

PowerShell:

```powershell
cd psu-ext-software\psu-fe
npm.cmd install
npm.cmd run dev
```

Bash:

```bash
cd psu-ext-software/psu-fe
npm install
npm run dev
```

Open the URL printed by Vite, normally `http://localhost:5173`.

PowerShell may prevent the `npm.ps1` shim from running. Using `npm.cmd`, as in
the example above, avoids changing the PowerShell execution policy.

### 3. Connect the frontend to the Proxy

1. Open **Config** in the frontend header.
2. In the **WebSocket** widget, enter:
   - Scheme: `ws`
   - Host: `localhost`
   - Port: `8080`
   - Path: `/ws/scpi`
3. Select **Connect**. Optionally enable **Proxy auto-connect** for later page
   loads.
4. In **Device List**, select **Add Device**, enter the TCP or USB profile, and
   save it.
5. Select **Connect** for the device. A successful connection triggers an
   identity query, and the device row should show its connection state and
   identity when the device supports `*IDN?`.

For a USB profile, complete the optional jSerialComm platform setup before
connecting the device.

Use `wss` only when the Proxy is actually served behind TLS. For a Proxy on a
different machine, replace `localhost` with that machine's hostname or IP and
ensure port `8080` is reachable from the browser.

At this point, the Proxy and frontend provide the minimal working PSU-EXT
dashboard configuration. IDE backend features, including script management and
execution, require the Script Runner configured in the next steps.

### 4. Start the Script Runner

Open another terminal in `psu-ext-software/psu-be` and start the Script Runner.

PowerShell:

```powershell
cd psu-ext-software\psu-be
.\mvnw.cmd -pl psu-be-script-runner -am spring-boot:run
```

Bash:

```bash
cd psu-ext-software/psu-be
./mvnw -pl psu-be-script-runner -am spring-boot:run
```

Leave this terminal running. The Script Runner listens on port `8081`. Verify
it in a browser or with a command-line request:

```text
http://localhost:8081/actuator/health
```

PowerShell:

```powershell
Invoke-RestMethod http://localhost:8081/actuator/health
```

Bash:

```bash
curl http://localhost:8081/actuator/health
```

The device API is `http://localhost:8081/api/devices`, and the script API is
`http://localhost:8081/api/scripts`.

By default, the Script Runner accepts frontend requests from
`http://localhost:5173`. If the frontend is served from another origin, update
the Script Runner configuration:

```yaml
psu:
  web:
    cors:
      allowed-origin: http://frontend-host:5173
```

The equivalent Spring environment variable is
`PSU_WEB_CORS_ALLOWED_ORIGIN`.

### 5. Configure Script Runner for IDE features

1. Open **IDE** in the frontend.
2. Open **Settings → Backend**.
3. Enter:
   - Scheme: `http`
   - Host: `localhost`
   - Port: `8081`
   - Root path: leave empty
4. Select **Test Connection**. The result should be **Connected**.
5. Select **Save**.
6. Open **Settings → Devices** to add, edit, connect, or disconnect Script
   Runner device profiles.

If the Script Runner runs on another machine, its CORS allowed origin must
match the browser frontend's full origin, including scheme and port.

### 6. Verify the complete setup

Use this startup order on subsequent runs:

1. Start the Proxy and wait for it to listen on port `8080`.
2. Start the frontend and connect it to the Proxy.
3. Add and connect a device through **Config → Device List**, then verify its
   connection state or `*IDN?` identity. The dashboard is now ready to use.
4. When IDE features are needed, start the Script Runner and verify
   `/actuator/health` on port `8081`.
5. Test and save the Script Runner backend in **IDE → Settings → Backend**.
6. Add and connect a Script Runner device, then run a simple query from the IDE.

A minimal Script Runner source is:

```javascript
(() => {
  const identity = query("PSU1", "*IDN?");
  log("INFO", identity);
  return { identity };
})();
```

Replace `PSU1` with the configured Script Runner device name and connect that
device before running the script.

### Optional setup

The steps below are not required for a TCP-only setup using device profiles
created through the frontend.

#### USB connections: select the jSerialComm platform

Complete this step only when the Proxy or Script Runner will connect to a USB
CDC serial device. Set `psu.transport.usb.platform` to the platform directory
that matches the operating system and CPU architecture. This value is not
detected automatically by PSU-EXT.

| Operating system | Supported values |
| --- | --- |
| Windows | `x86_64`, `x86`, `aarch64`, `armv7` |
| macOS | `x86_64`, `x86`, `aarch64` |
| Linux | `x86_64`, `x86`, `armv8_64`, `armv7hf` |

Most 64-bit Intel and AMD Windows or Linux PCs use `x86_64`. Apple Silicon Macs
use `aarch64`. On Linux, `uname -m` can help identify the architecture.

Set the value in each terminal that starts a backend that will use USB, before
running its Maven start command.

PowerShell:

```powershell
$env:PSU_TRANSPORT_USB_PLATFORM = "x86_64"
```

Bash (macOS or Linux):

```bash
export PSU_TRANSPORT_USB_PLATFORM=x86_64
```

Alternatively, set the property directly in the relevant backend configuration
file:

- Proxy: `psu-be-proxy/src/main/resources/application.yml`
- Script Runner: `psu-be-script-runner/src/main/resources/application.yaml`

```yaml
psu:
  transport:
    usb:
      platform: x86_64
```

The backend uses this value to stage the native jSerialComm library for the
current operating system before opening a serial port. A wrong value commonly
causes a native-library loading or resource-not-found error. If the property is
unset, PSU-EXT skips this staging step and leaves native-library discovery to
jSerialComm, which may not work consistently on every machine.

On Linux, the user must also have permission to open the device, commonly
`/dev/ttyACM0` or `/dev/ttyUSB0`. Distribution-specific groups are often named
`dialout` or `uucp`; log out and back in after adding group membership.

#### Predefine device registries

The normal setup is to create device profiles through the frontend after the
services start:

- Use **Config → Device List** for Proxy devices.
- Use **IDE → Settings → Devices** for Script Runner devices.

To predefine profiles before startup, edit the configured device-registry JSON
file. Both services default to `./devices.json` when started from `psu-be`. A
missing registry file is created automatically as an empty JSON array.

| Field | Meaning |
| --- | --- |
| `id` | Unique numeric device ID in the range `0` to `255` |
| `name` | Unique uppercase alphanumeric name, up to eight characters |
| `type` | `TCP` or `USB` |
| `ip` | TCP host or IP address; `null` for USB |
| `port` | TCP port as text, or the serial-port name |
| `baudrate` | USB baud rate; `null` for TCP |

Example registry with one TCP and one USB device:

```json
[
  {
    "id": 0,
    "name": "PSU1",
    "type": "TCP",
    "ip": "192.0.2.10",
    "port": "5025",
    "baudrate": null
  },
  {
    "id": 1,
    "name": "USBPSU",
    "type": "USB",
    "ip": null,
    "port": "COM3",
    "baudrate": 115200
  }
]
```

Replace `192.0.2.10`, `COM3`, and the baud rate with values for the actual
device. On macOS or Linux, use the complete serial device name shown by the
operating system instead of `COM3`. USB profiles also require the optional
jSerialComm platform setup above.

The Proxy and Script Runner may share `./devices.json` when they should use the
same profiles. Do not edit a shared registry through both running services at
the same time. If the services should maintain independent profiles, give each
one a separate file before starting it.

PowerShell, in the Proxy terminal:

```powershell
$env:PSU_CONNECTION_DEVICES_FILE = "./proxy-devices.json"
```

PowerShell, in the Script Runner terminal:

```powershell
$env:PSU_CONNECTION_DEVICES_FILE = "./script-runner-devices.json"
```

Bash, in the corresponding terminals:

```bash
export PSU_CONNECTION_DEVICES_FILE=./proxy-devices.json
```

```bash
export PSU_CONNECTION_DEVICES_FILE=./script-runner-devices.json
```

### Troubleshooting

| Symptom | Likely cause and action |
| --- | --- |
| `Port 8080 was already in use` | Another process is using the Proxy port. Stop it or override `server.port`, then use the same port in the frontend WebSocket configuration. |
| `Port 8081 was already in use` | Another process is using the Script Runner port. Stop it or override `server.port`, then update the frontend Script Backend port. |
| Browser reports a CORS error | Set `psu.web.cors.allowed-origin` or `PSU_WEB_CORS_ALLOWED_ORIGIN` to the frontend's exact origin and restart Script Runner. |
| WebSocket remains disconnected | Confirm the Proxy is running, and verify scheme, host, port `8080`, and path `/ws/scpi`. Also check firewall and TLS settings for remote hosts. |
| Script Backend test calls port `8080` or fails immediately | Script Runner uses `8081` by default. Open **IDE → Settings → Backend**, set port `8081`, test, and save. |
| jSerialComm native library cannot load | Set `PSU_TRANSPORT_USB_PLATFORM` to one of the exact values for the current OS and architecture, then restart the backend. |
| Serial port is missing, busy, or cannot be opened | Close other serial applications, verify the COM/TTY name, reconnect the device, and check Linux group/device permissions. A single serial port cannot normally be opened by both backends simultaneously. |
| Device connects but queries time out | Verify the device baud rate or TCP endpoint and confirm its SCPI line ending and response delimiter match the backend configuration. |
| Changes appear in one backend but not the other | Each service loads and owns its registry state. Restart the other service after external edits, or use separate device-registry files and manage each through its own UI. |
