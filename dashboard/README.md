# MeshCore MQTT Dashboard (PyScript)

A browser-based real-time monitoring dashboard for MeshCore LoRa repeaters.  
All data-processing logic is written in **Python running directly in the browser** via [PyScript](https://pyscript.net/) (powered by Pyodide/WebAssembly). No server or Python installation is required to run the dashboard.

## Features

| Feature | Details |
|---|---|
| **Battery telemetry** | Live voltage (mV), charge-level bar, history chart |
| **External power monitor** | INA3221 channel voltages / currents (when `ext_power_type = "ina3221"` in config) |
| **Signal quality** | RSSI / SNR per-packet display + scrolling history chart |
| **Radio airtime** | TX / RX airtime seconds and lifetime utilisation % |
| **Device stats** | Noise floor, RX errors, queue length, device uptime |
| **Live packet feed** | Scrolling table of RX/TX packets (direction, route, type, hash, path) |
| **Multi-broker presets** | One-click fill for LetsMesh US / EU or a local broker |
| **Pure Python UI logic** | All JSON parsing, state management, and DOM updates use PyScript — no JavaScript business logic |

## Architecture

```
Browser
  │
  ├─ mqtt.js (CDN)          WebSocket MQTT client
  │     │  on message()
  │     └──────────────────► PyScript (Python / Pyodide)
  │                               parse JSON
  │                               update DOM
  │                               push to Chart.js
  │
  ├─ Chart.js (CDN)         Battery & signal charts
  └─ PyScript (CDN)         Python 3.11 runtime (Pyodide / WASM)
```

The Python code in `index.html` reuses the same JSON field names produced by
the `bridge/` package:

| MQTT topic suffix | Key fields consumed |
|---|---|
| `/status` | `origin`, `origin_id`, `model`, `firmware_version`, `radio`, `status`, `stats.battery_mv`, `stats.battery_source`, `stats.uptime_secs`, `stats.tx_air_secs`, `stats.rx_air_secs`, `stats.noise_floor`, `stats.recv_errors`, `stats.extpower` |
| `/packets` | `direction`, `route`, `packet_type`, `RSSI`, `SNR`, `score`, `hash`, `time`, `origin`, `path` |
| `/debug` | `message` |

## Quick Start

The dashboard is a **single self-contained HTML file** with no build step.

### Open locally

```bash
# From the repo root:
open dashboard/index.html            # macOS
xdg-open dashboard/index.html        # Linux
start dashboard/index.html           # Windows
```

> **Note:** Some browsers block WebSocket connections from `file://` URLs.
> Use a local HTTP server if you hit this restriction:
>
> ```bash
> python3 -m http.server 8080 --directory dashboard
> # then open http://localhost:8080
> ```

### Serve with Docker (alongside the bridge)

Add a static file server to your `docker-compose.yml`:

```yaml
services:
  dashboard:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./dashboard:/usr/share/nginx/html:ro
```

Then open `http://<host>:8080`.

## Connecting to a Broker

1. **Broker URL** — must be a WebSocket URL:
   - LetsMesh US: `wss://mqtt-us-v1.letsmesh.net:443/mqtt`
   - LetsMesh EU: `wss://mqtt-eu-v1.letsmesh.net:443/mqtt`
   - Local Mosquitto (WebSocket port): `ws://localhost:9001`

2. **Topic Filter** — subscribes to everything under the prefix:
   - `meshcore/#` — all nodes, all regions
   - `meshcore/SEA/#` — all nodes in Seattle region
   - `meshcore/SEA/ABCD1234.../#` — one specific node

3. **Username / Password** — for `password` auth brokers.  
   For LetsMesh token auth, enter your `v1_<PUBKEY>` username and JWT token.

Click **Connect**. Data starts appearing immediately when your repeater
publishes status or packet messages.

## PyScript Details

The dashboard uses **PyScript 2024.11.1** loaded from the official CDN:

```html
<link  rel="stylesheet" href="https://pyscript.net/releases/2024.11.1/core.css">
<script type="module"   src="https://pyscript.net/releases/2024.11.1/core.js"></script>
```

Python code runs in a `<script type="py">` block.  
`from js import window, document` gives Python direct access to browser APIs.  
`from pyscript.ffi import create_proxy` wraps Python callbacks so JavaScript
(mqtt.js) can call them when messages arrive.

No packages need to be installed via `micropip` — the dashboard uses only
Python standard-library modules (`json`, `datetime`).

## Battery Voltage Interpretation

The charge-level bar and colour use a simple Li-ion/LiPo model:

| Voltage | Level |
|---|---|
| ≥ 4100 mV | > 75% — green |
| 3700 – 4100 mV | 25–75% — green |
| 3400 – 3700 mV | 10–25% — amber |
| < 3400 mV | < 10% — red |

The raw mV reading is always shown regardless of the bar estimate.

## External Power Monitor (INA3221)

When `ext_power_type = "ina3221"` is set in `config.toml`, the bridge sends
an `extpower` sub-object inside the `stats` field of each status message:

```json
{
  "stats": {
    "battery_mv": 3780,
    "battery_source": "extpower ch2",
    "extpower": {
      "ch1_voltage_mv": 5012,
      "ch2_voltage_mv": 3780,
      "ch3_voltage_mv": 12050,
      "ch1_current_ma": 142
    }
  }
}
```

The dashboard renders each channel as its own tile in the
**External Power Monitor** section, which is hidden until this data arrives.

## Requirements

- A modern browser (Chrome 90+, Firefox 90+, Edge 90+, Safari 15+)
- Internet access to the PyScript, Chart.js, and mqtt.js CDNs on first load  
  (or serve the CDN assets locally for air-gapped deployments)
- An MQTT broker accessible via WebSocket (`ws://` or `wss://`)
