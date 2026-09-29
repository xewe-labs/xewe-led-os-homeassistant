# XeWe LED — Home Assistant integration for XeWe LED OS

Personal project (XeWe Labs) · 2026-07-08 → 2026-09-14 · Solo: Max Dokukin · Status: Ongoing (integration 0.1.0; firmware side shipped in XeWe LED OS 2.3.0)

## Overview

This repository connects an ESP32 running [XeWe LED OS](https://github.com/xewe-labs/xewe-led-os) to Home Assistant with
a hands-off pairing flow: the device announces itself on the network, Home Assistant shows it under **Discovered**, you
click **Submit**, and the light appears. You never type a broker address or password — the integration forwards Home
Assistant's own MQTT credentials to the device. It ships two parts: a Home Assistant **custom integration**
(`custom_components/xewe_led_os/`, installed through HACS) that discovers the device over zeroconf/mDNS and posts the
broker credentials to the device's `/provision` endpoint, and a **sample ESP32 sketch** that publishes the entities by
itself through MQTT discovery. The same contract is implemented by the `HomeAssistant` module of XeWe LED OS
(`src/Modules/Software/SmartHome/HomeAssistant/`), which the staged copy in `firmware/xewe_led_os/integration/` was
written for.

## Highlights

- Zero-typing pairing: mDNS service `_xewe-led-os._tcp` → zeroconf config flow → `POST /provision {host, port, user, pass}` → MQTT (`config_flow.py`, `manifest.json`)
- Entities come from the firmware's own retained MQTT discovery messages — a JSON-schema `light` (on/off, brightness, HS colour), a `select` for the mode and `number` entities for the current mode's parameters; there is no platform file in the integration (`AGENTS.md`)
- Broker-host substitution: when Home Assistant stores its broker as `localhost` / `core-mosquitto`, the flow hands the device a LAN-reachable address instead (`LOCAL_BROKER_HOSTS`, `_suggested_broker_host`)
- Clean removal: deleting the device in Home Assistant calls `POST /deprovision`, which clears every retained discovery topic so no ghost entities remain (`__init__.py`)
- 46 commits, 45 of them in July 2026 (`git log`); 1,577 lines in total (Python 269, sample sketch 685, staged firmware module 623)

## How it works

```
ESP32 device                              Home Assistant
─────────────                             ──────────────
connects to your Wi-Fi
you press 'y' → discovery mode            
announces _xewe-led-os._tcp (mDNS) ─────►  appears under "Discovered"
                                          you click Configure → Submit
receives broker address + login   ◄─────  POST /provision with HA's MQTT details
connects, publishes retained      ─────►  light + mode select + mode params are created
discovery config and state
```

You do this once. After that the device remembers the broker (in NVS) and reconnects by itself on every reboot.

- **Integration** (`custom_components/xewe_led_os/`) — `config_flow.py`: `zeroconf` step (reads `mac` and `name` from the TXT record, one entry per MAC), manual `user` step (enter the device IP), `pair` step (confirm, then provision with a 10 s timeout); `__init__.py`: setup/unload and `/deprovision` on removal; `const.py`: domain `xewe_led_os`, endpoints, local-only broker hosts; `brand/` icons (256 and 512 px).
- **Sample firmware** (`firmware/xewe_led_os/xewe_led_os.ino`) — a mock LED data model (modes Solid, Fade, Rainbow; Fade exposes speed + depth, Rainbow speed) that mirrors the real device: device id `xewe_led_os_<mac>`, topics under `xewe_led_os/<device_id>/…`, discovery under the `homeassistant/` prefix, broker credentials in NVS (`Preferences`), MQTT buffer 1,024 bytes.
- **Staged firmware module** (`firmware/xewe_led_os/integration/HomeAssistant.{h,cpp}`) — the real-device version that sources all state from XeWe LED OS and fans changes out through the `sync_*` hooks; it does not compile inside this repository (its headers live in xewe-led-os) — that is expected.

Naming rule: the user-facing display name is **XeWe LED**; everything internal stays `xewe_led_os` (domain, directory,
mDNS service, MQTT topics, device id prefix).

## Results

| Metric | Value | Baseline / note |
|---|---|---|
| Entities per device | 1 light + 1 mode select + the current mode's number entities | created by firmware MQTT discovery |
| Integration version | 0.1.0 (`manifest.json`) | `iot_class: local_push`, depends on `mqtt` |
| Home Assistant versions | ≥ 2024.1.0 (`hacs.json`); docs target 2026.2+, local brand icons need 2026.3+ | `AGENTS.md` |
| Code | 269 lines of Python, 685-line sample sketch, 623-line staged module | raw `wc -l` |

No automated tests; validation is `python -m py_compile` on the integration and a JSON parse of the manifests (see `AGENTS.md`).

## Getting started

Quickstart: install MQTT → install HACS → install this integration → flash the firmware → pair.

### Step 1 — Install MQTT

The device reports to Home Assistant over **MQTT**, so MQTT has to be running before you pair. If you already have MQTT set up, skip to Step 2.

**Home Assistant OS / Supervised** (has the App Store):

1. Go to **Settings → Devices & Services**.
2. Click **+ Add integration**, search for **MQTT**, and pick **MQTT**.
3. Choose **"Use the official Mosquitto MQTT Broker app"** — Home Assistant installs and starts the broker for you.
4. Verify it's running: **Settings → Apps** → **Mosquitto broker** is listed and started.

**Home Assistant Container / Core** (no App Store): run your own broker first (e.g. the `eclipse-mosquitto` Docker image
on port `1883`), then add the **MQTT** integration under **Settings → Devices & Services → + Add integration → MQTT**
and point it at that broker's IP address.

### Step 2 — Install HACS

This integration is not built into Home Assistant, so you install it through **HACS** (Home Assistant Community Store).
If you don't already have HACS, follow the official install guide: https://www.hacs.xyz/docs/use/download/download/

### Step 3 — Install this integration via HACS

1. Open **HACS** in the Home Assistant sidebar.
2. Click the **three-dot menu** (top right) → **Custom repositories**.
3. Paste `https://github.com/xewe-labs/xewe-led-os-homeassistant`, set **Type / Category** to **Integration**, and click **Add**.
4. Find **XeWe LED** in the list, open it, and click **Download**.
5. **Restart Home Assistant** (Settings → System → Restart).

> **Without HACS:** copy the [`custom_components/xewe_led_os/`](custom_components/xewe_led_os) folder into your Home
> Assistant config folder (the one with `configuration.yaml`) so the path is `<config>/custom_components/xewe_led_os/`,
> then restart. Everything else works the same; you just won't get HACS update notifications.

### Step 4 — Flash the firmware

- **XeWe LED OS** (the real device): flash release 2.3.0 or later from https://maxdokukin.com/projects/xewe-led-os; the
  Home Assistant module is set up on first boot (or later with `$homeassistant enable`) and prints its mDNS name and the
  manual `/provision` fallback on the serial console.
- **Sample sketch** (`firmware/xewe_led_os/xewe_led_os.ino`): in the Arduino IDE install **PubSubClient** (knolleary) and
  **ArduinoJson** (bblanchon) (`WiFi.h`, `ESPmDNS.h`, `WebServer.h` and `Preferences.h` come with the ESP32 core). Create
  `firmware/xewe_led_os/wifi_c.h` (gitignored) defining `WIFI_SSID` and `WIFI_PASS`, set `LED_PIN` (default 8) and
  `LED_ACTIVE_HIGH` at the top of the sketch, flash it and open the Serial Monitor at **115200 baud**.

The device must be on the **same network** as Home Assistant.

### Step 5 — Pair the device

1. In the Serial Monitor the device reports it joined Wi-Fi. On the sample sketch, **press `y`** to enter discovery mode; it stays on until the device is provisioned.
2. In Home Assistant, go to **Settings → Devices & Services**. The device appears under **Discovered**. Click **Configure**, then **Submit**.
3. Home Assistant sends the device its MQTT details; the device connects and publishes its light, mode select and mode parameters.

If discovery does not show the device, add the integration manually (**+ Add integration → XeWe LED**) and enter the device IP,
or POST the broker details yourself:

```bash
curl -X POST http://<device-ip>/provision \
  -d '{"host":"<mqtt-host>","port":1883,"user":"<mqtt-user>","pass":"<mqtt-pass>"}'
```

> **If pairing succeeds but the device can't connect to MQTT:** Home Assistant sometimes stores its broker address as
> `core-mosquitto` or `localhost`, which the ESP32 can't reach. The flow substitutes the LAN address Home Assistant uses
> to reach the device; if your broker runs on a different machine, provision manually as above.

### For developers

```bash
python -m py_compile custom_components/xewe_led_os/*.py
python -c "import json; [json.load(open(p)) for p in ['hacs.json','custom_components/xewe_led_os/manifest.json','custom_components/xewe_led_os/strings.json','custom_components/xewe_led_os/translations/en.json']]"
```

Keep `strings.json` and `translations/en.json` identical in structure. Guidance for coding agents: [AGENTS.md](AGENTS.md).

## Documents

- [AGENTS.md](AGENTS.md) — architecture invariants, naming rule, firmware gotchas, validation
- [Integration manifest](custom_components/xewe_led_os/manifest.json) · [HACS metadata](hacs.json)
- Firmware side: [XeWe LED OS](https://github.com/xewe-labs/xewe-led-os) (`src/Modules/Software/SmartHome/HomeAssistant/`)
- Project page: https://maxdokukin.com/projects/xewe-led-os

| Path | What it is |
| --- | --- |
| `custom_components/xewe_led_os/` | The Home Assistant integration: discovers the device over mDNS and hands it the broker credentials |
| `firmware/xewe_led_os/xewe_led_os.ino` | Sample ESP32 firmware with a mock LED model: pairs and exposes a light, a mode select and mode parameters |
| `firmware/xewe_led_os/integration/` | The real-device Home Assistant module, staged for XeWe LED OS |
| `hacs.json` | HACS metadata |

## Scope

The integration only performs discovery and the credential handoff; the firmware owns the entities. MQTT is a
documented prerequisite and is not checked in code. A dedicated per-device broker login and an on-device pairing
display are left as follow-ups.
