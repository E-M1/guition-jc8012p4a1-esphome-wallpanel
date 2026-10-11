# Home Assistant on a 10.1" ESP32-P4 Wall Panel 

**Guition JC8012P4A1 · ESPHome · RemoteWebViewServer — complete working guide**

# ESPHome + RemoteWebView: Home Assistant Lovelace on a Guition JC8012P4A1 (ESP32-P4) wall panel 10.1"
Turn a cheap Chinese 10.1" touch panel into a beautiful, fully interactive
Home Assistant Lovelace dashboard wall panel.


## What this is

A working, tested setup that shows a **Home Assistant Lovelace view** on a cheap
10.1" ESP32-P4 touch panel. The panel itself stays simple: ESPHome firmware that
receives streamed JPEG frames of a browser page and forwards touch input back.
All the heavy lifting (the browser rendering the Lovelace UI) runs in Docker on
any machine on your LAN.

**Performance:** with the tweaks in this config it runs *smooth* — no graphical
artefacts of any kind. Even a live **Frigate camera stream** plays fine on the
panel. Touch response is faster than my old android tablet...

## Hardware this was built on

| | |
|---|---|
| Manufacturer | Espressif (ESP32 Brookesia board) |
| SKU / batch | 10153002-V3, batch 2638 |
| Model | JC8012P4A1V_l8W_Y1 |
| Chip | ESP32-P4, revision v3.2 (+ ESP32-C6 co-processor for WiFi) |
| Display | 10.1" 800×1280 MIPI-DSI (JD9365 driver), GSL3680 touch controller |

**Panel revision warning:** the same product name ships with different display
panels. ESPHome's `mipi_dsi` component has two models: `JC8012P4A1` and
`JC8012P4A1-V2`. Check the batch sticker on the back — batch **≥ 2624** needs
`JC8012P4A1-V2` (which this config uses). Wrong driver = 3 solid color bars and
nothing else.

## How it works

1. **RemoteWebViewServer** (Docker, on any LAN machine) launches headless
   Chromium, loads your Lovelace URL, and streams JPEG tiles of the page over a
   WebSocket (port 8085).
2. **The panel** (this ESPHome config) connects, hardware-decodes the JPEG
   frames (P4 JPEG decoder + 32 MB PSRAM), and paints them to the display.
3. **Touch** (GSL3680) events are streamed back and injected into the browser —
   the page is fully interactive.

## Credits — who did what

I wrote none of the code; I only assembled and documented a working setup for
my specific hardware. All credit goes to:

- **strange-v** — RemoteWebViewClient ESPHome component:
  https://github.com/strange-v/RemoteWebViewClient · RemoteWebViewServer Docker
  image: https://hub.docker.com/r/strangev/remote-webview-server · hardware JPEG
  decode: https://github.com/strange-v/jpegdec-esphome
- **jtenniswood** — guition-esp32-p4-jc8012p4a1 (working config & panel
  revision knowledge): https://github.com/jtenniswood/guition-esp32-p4-jc8012p4a1
- **kvj** — early GSL3680 / JD9365 work:
  https://github.com/kvj/esphome/tree/jd9365_gsl3680
- **ESPHome project** — mipi_dsi display component and docs:
  https://esphome.io/components/display/mipi_dsi
- Home Assistant community threads on the JC8012P4A1 for collected debugging wisdom.

If any platform name is rejected by the compiler, cross-check jtenniswood's
`guition-esp32-p4-jc8012p4a1/esphome.yaml` and copy the differing block.

## Project layout (on the machine doing the compiling)
PC: Latest Linux Mint, so anything debian based should be fine
Server Debian latest

```
~/jc8012-panel/                         
|
|-- config/
|   |-- secrets.yaml               
|   |-- jc8012-panel.yaml          full device config (in this README, below)
|   |-- my_components/
|       |-- gsl3680/               locally patched touch driver (required!)
|           |-- __init__.py
|           |-- touchscreen.py
|           |-- gsl3680.h
|           |-- gsl3680.cpp
|           |-- gsl_point_id.h
|           |-- gsl_point_id.cpp
|           +-- gsl3680_firmware.h
|
+-- README.md                      this file
```
The `my_components/gsl3680/` folder is **not optional**: the device config loads
the touch driver from it (`source: my_components` in the YAML). Use the patched
files, not the stock GSL3680 component — the touch transform/behaviour was
fixed for this panel.

Output lands in
`config/.esphome/build/jc8012-panel/`:
`factory.bin` = first USB flash, `firmware.bin`

### secrets.yaml (create it yourself, not included here)

```yaml
wifi_ssid: "YourWifi"
wifi_password: "YourPassword"
```

---

## STEP 1 — Install ESPHome (Python venv)

```bash
sudo apt install python3-venv
mkdir -p ~/jc8012-panel && cd ~/jc8012-panel
python3 -m venv venv
source venv/bin/activate
pip install esphome
```

Every time you come back to this project: `cd ~/jc8012-panel && source venv/bin/activate`.

## STEP 2 — Create the config

Save the YAML below as `~/jc8012-panel/config/jc8012-panel.yaml`, and edit the
two `CHANGE ME` substitutions:

```yaml
substitutions:
  name: jc8012-panel
  friendly_name: Wall Panel
  rwv_server: "192.168.1.100:8085"                     # CHANGE ME (machine running RemoteWebViewServer)
  rwv_url: "http://192.168.1.100:8123/lovelace/tabletview"  # CHANGE ME

esphome:
  name: ${name}
  friendly_name: ${friendly_name}
  min_version: 2026.1.0

esp32:
  variant: ESP32P4
  cpu_frequency: 400MHz
  framework:
    type: esp-idf
    advanced:
      enable_idf_experimental_features: true
    components:
      - name: "espressif/esp_websocket_client"
        ref: 1.5.0
      - name: "bitbank2/jpegdec"
        source: https://github.com/strange-v/jpegdec-esphome
        ref: main

esp32_hosted:
  variant: ESP32C6
  active_high: true
  clk_pin: GPIO18
  cmd_pin: GPIO19
  d0_pin: GPIO14
  d1_pin: GPIO15
  d2_pin: GPIO16
  d3_pin: GPIO17
  reset_pin: GPIO54

esp_ldo:
  - channel: 3
    voltage: 2.5V

psram:
  speed: 200MHz

external_components:
  - source: github://strange-v/RemoteWebViewClient@main
    refresh: 0s
    components: [remote_webview]
  # gsl3680 — local patched copy, loaded from config/my_components/gsl3680
  - source: my_components
    components: [gsl3680]

logger:
  baud_rate: 115200
  level: warn
  hardware_uart: UART0

api:
  reboot_timeout: 0s

ota:
  platform: esphome

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "${name} setup"

captive_portal:

i2c:
  - id: i2c_bus
    sda: GPIO7
    scl: GPIO8
    frequency: 400kHz
    scan: true

display:
  - platform: mipi_dsi
    model: JC8012P4A1-V2
    id: my_display
    auto_clear_enabled: false

touchscreen:
  - platform: gsl3680
    id: my_touch
    reset_pin: GPIO22
    interrupt_pin: GPIO21
    update_interval: 100ms
    transform:
      swap_xy: false
      mirror_x: true
      mirror_y: false

# Backlight enable pin (JC8012P4A1 backlight = GPIO23) — PWM so HA can dim it
output:
  - platform: ledc
    pin: GPIO23
    id: backlight_pwm
    frequency: 1000 Hz
    min_power: 0.03
    zero_means_zero: true

light:
  - platform: monochromatic
    name: "Panel Backlight"
    id: panel_backlight
    output: backlight_pwm
    restore_mode: RESTORE_DEFAULT_ON

remote_webview:
  id: rwv1
  touchscreen_id: my_touch
  server: ${rwv_server}
  url: "self-test"        # STEP 9: switch to ${rwv_url}
  jpeg_quality: 80
  min_frame_interval: 90
  max_bytes_per_msg: 61440
  full_frame_tile_count: 1
```

## STEP 3 — Validate

```bash
esphome config config/jc8012-panel.yaml
```

Fix any errors it names until it passes.

## STEP 4 — Compile

```bash
esphome compile config/jc8012-panel.yaml
```

First run: 1–3 GB toolchain download, 10–20 minutes, one time only.
Afterwards: 2–5 minutes. Ends with "Creating BIN file".

```bash
ls -lh config/.esphome/build/jc8012-panel/ | grep -i bin
```

The biggest `.bin` is `firmware.factory.bin` — that's your flash image.

## STEP 5 — USB flash

Hold **BOOT**, plug in USB, tap **RESET**, release **BOOT** (see photos for
button placement). Then:

```bash
ls /dev/ttyACM* /dev/ttyUSB*
# nothing? → sudo usermod -a -G dialout $USER, log out/in (maybe reboot)
esptool.py --port /dev/ttyACM0 --baud 921600 --chip esp32p4 write_flash 0x0 \
  config/.esphome/build/jc8012-panel/firmware.factory.bin
```

Fails? Check whether it's `ttyACM*` or `ttyUSB*` on your machine, then retry
the button dance: unplug, hold BOOT, plug in, tap RESET, release BOOT — or
plugged in, hold BOOT, tap RESET, release BOOT after 2 seconds.

## STEP 6 — First boot

The panel joins Wi-Fi (test with `ping jc8012-panel.local` or its IP) and shows
a **self-test pattern (RGB bars with in white letters R G B inside of the color bars)** — that means the screen works. Watch it:

```bash
esphome logs config/jc8012-panel.yaml --device /dev/ttyACM0
# (or /dev/ttyUSB0, whichever exists on your machine)
```

## STEP 7 — Run RemoteWebViewServer (Docker)

### Install Docker (Debian/Ubuntu)

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER   # log out/in afterwards
```

### Create the compose file

```bash
mkdir -p ~/rwvserver/pw-data   # writable browser profile dir
nano ~/rwvserver/docker-compose.yml
```

Paste this (edit `INJECT_JS_URL` — see STEP 8):

```yaml
rwvserver:
  image: strangev/remote-webview-server:latest  # use :beta for pre-release
  container_name: remote-webview-server
  restart: unless-stopped
  environment:
    TILE_SIZE: 64 #32
    FULL_FRAME_TILE_COUNT: 4
    FULL_FRAME_AREA_THRESHOLD: 0.5
    FULL_FRAME_EVERY: 50
    EVERY_NTH_FRAME: 1
    MIN_FRAME_INTERVAL_MS: 90
    JPEG_QUALITY: 80
    MAX_BYTES_PER_MESSAGE: 61440
    WS_PORT: 8085
    DEBUG_PORT: 9221 # internal debug port
    HEALTH_PORT: 18080
    PREFERS_REDUCED_MOTION: false
    INJECT_JS_URL: ""            # optional JS to inject into the page, see STEP 8
    INJECT_JS_ALLOW_HTTP: false
    USER_DATA_DIR: /pw-data
    BROWSER_LOCALE: "en-US"
  ports:
    - "8085:8085"                # WebSocket stream
    - "9222:9222"                # external DevTools via socat
  expose:
    - "18080"                    # health endpoint (internal)
    - "9221"                     # internal DevTools port
  volumes:
    - ./pw-data:/pw-data
  shm_size: 1gb
  healthcheck:
    test: ["CMD-SHELL", "curl -fsS http://localhost:18080 || exit 1"]
    interval: 10s
    timeout: 3s
    retries: 5
    start_period: 10s

debug-proxy:
  image: alpine/socat
  container_name: remote-webview-server-debug
  restart: unless-stopped
  network_mode: "service:rwvserver"
  depends_on:
    rwvserver:
      condition: service_healthy
  command:
    - "-d"
    - "-d"
    - "TCP-LISTEN:9222,fork,reuseaddr,keepalive" # external DevTools port
    - "TCP:127.0.0.1:9221"
```

### Start and verify

```bash
cd ~/rwvserver
docker compose up -d
docker compose logs -f          # Ctrl+C to stop following
ss -tlnp | grep 8085            # confirm it's listening
```

## STEP 8 — Log the browser in (brave://inspect)

Lovelace normally requires a login, but the server's browser is headless — no
keyboard. Use Chrome/Brave DevTools remote debugging:

1. Open **Brave** (or Chrome) on your PC and go to `brave://inspect/#/devices`
   (`chrome://inspect/#/devices` in Chrome).
2. Click **Configure…** and add `SERVER_IP:9222` (the machine running the
   Docker server, e.g. `192.168.1.100:9222`).
3. The headless browser's page appears under "Remote Target" → click **inspect**.
4. A DevTools window opens showing the panel's page — you can click around it.
   Navigate to your Home Assistant login page and **type your HA username and
   password into that DevTools-rendered page** (the profile is persistent in
   `./pw-data`, so the login sticks across reboots).
5. Once logged in and your Lovelace view is on screen, close DevTools.


## STEP 9 — Go live

Change `url: "self-test"` → `url: ${rwv_url}` in the config, rebuild, and
either re-flash over USB as in STEP 5, or let ESPHome do it:

```bash
esphome compile config/jc8012-panel.yaml
esphome run config/jc8012-panel.yaml
```

Your Lovelace view now appears on the panel, fully touch-interactive.

## STEP 10 — Adopt in Home Assistant (optional)

Settings → Devices & services → ESPHome → add the panel's IP → enable
**"Allow the device to perform Home Assistant actions"**.

The panel then appears as a device with the **Panel Backlight** light — turn
the screen off/on or dim it. I use this in a script that ties brightness to an
outdoor lux sensor.

---

## Troubleshooting

- **JPEGDEC linker errors** → use `client@main` + ESPHome ≥ 2026.1.0
- **No serial port found** → `dialout` group or a charge-only USB cable
- **Black screen** → display pins vs. jtenniswood's config; check panel
  revision (V2 model!)
- **3 color bars without letter inside** → wrong `mipi_dsi` model for your panel revision
- **RGB bars persist after everything is up** → panel is fine; check the
  websocket link to the server (`docker compose ps`, port 8085, firewall)
  use cli to follow boot of esp device: esphome -logs jc8012-panel.yaml --device /dev/ttyUSB0-
- **`transport_ws: Error connecting to host ...` in logs** → server unreachable;
  check `rwv_server`, that Docker is up, and any firewall on 8085
- **No dashboard on screen** → server/firewall on 8085 (see above)
- **Camera-card style reboots** → lower `jpeg_quality` to ~70

## License

MIT.

