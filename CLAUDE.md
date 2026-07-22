# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

ESPHome firmware config for a single ESP8266-based 8-digit VFD (vacuum
fluorescent display) clock, integrated with Home Assistant. Built on
[Trip5's EspHome-VFD-Clock](https://github.com/trip5/EspHome-VFD-Clock)
template — one device, one YAML file (`vfdclock.yaml`), not a multi-device
fleet. Device name: `vfd-clock-ha` / friendly name `VFD-Clock-HA`.

This device is **severely memory-constrained** compared to `esphomeclock`
(a sibling project, ESP32-based) — it's an ESP8266 with `web_server` and
verbose `logger` levels deliberately disabled to save RAM/flash. Last known
build: RAM 51.7% (42312/81920 bytes), Flash 45.5% (465845/1023984 bytes).
Don't casually re-enable `web_server` or bump `logger:` above `WARN`
without checking the resulting RAM/flash headroom.

## Build / compile / deploy

Same dual-host setup as the `esphomeclock` sibling project — **iterate on
`.10`, promote to `.12` once confirmed working**:

- **`10.1.10.10`** (Unraid, Docker container `ESPHome`) — primary dev/test
  host, full CLI access via SSH. Local file: `\\10.1.10.10\esphome-config\vfdclock.yaml`.
- **`10.1.10.12`** (Home Assistant instance, "ESPHome Device Builder"
  add-on) — long-term/production copy, Ingress-only (no SSH, no exposed
  port). **Important:** on `.12` this same device is tracked under a
  *different filename*, `esphome/ehvclock-ha.yaml` (a legacy name from
  before this repo existed) — always push updates there as
  `ehvclock-ha.yaml`, never as `vfdclock.yaml`, or the dashboard will treat
  it as a second, duplicate device sharing the same `vfd-clock-ha` node
  name.

There is no per-device build-counter script for this device (unlike
`esphomeclock`'s `bump_fw_version_esphomeclock.py`) — no firmware-version
tracking is wired into this YAML.

```bash
# --- Dev/test on .10 (primary; do this first) ---
docker exec ESPHome esphome config /config/vfdclock.yaml     # validate
docker exec ESPHome esphome compile /config/vfdclock.yaml    # compile
docker exec ESPHome esphome upload /config/vfdclock.yaml --device vfd-clock-ha.local  # OTA upload
# (mDNS hostname is more reliable here than a fixed IP; the device is DHCP
#  and its address has moved before — last observed at 10.1.10.67)

# --- Promote to .12 (only after the .10 deploy above is confirmed working) ---
# Push the SAME vfdclock.yaml content to \\10.1.10.12\config\esphome\ehvclock-ha.yaml
# (note the filename change). Then in a browser: Home Assistant -> ESPHome
# sidebar panel -> the vfd-clock-ha/ehvclock-ha entry -> Validate -> Install.
```

Secrets are shared with `esphomeclock` (same `wifi1-3`, `ota_password`) —
this device additionally *does* use `esp_fallbackpass` for its captive-portal
fallback AP password (`esphomeclock` doesn't reference it, which is why that
project's docs call it "unused/unexplained" — it's used here). HA API
encryption (`encryption_key`) is commented out/unused for this device;
Home Assistant integration instead goes through a single templated sensor
(`sensor.vfd_clock_data`, see Architecture below), not the encrypted native API.

## Architecture

Single YAML file, `vfdclock.yaml` (~1100 lines), using two of Trip5's
external git components (`vfd` for the display driver, `rx8025` for the
RTC) plus stock ESPHome core components and `lambda:` blocks. Key pieces:

- **Display lambda** (`display: - platform: vfd`) — one large lambda,
  branching in priority order: boot warmup (first 5s after boot,
  `boot_warmup_until_ms`) → hourly pulse (brief alternating full/blank
  flash at the top of every hour, `hourly_pulse_until_ms`) → status cues
  (`status_cue_text`/`status_cue_until_ms`, used for transient "WIFI OK",
  "WIFI LOST", "TIME SYNC" messages) → HA push messages (via the `message`
  API service) → rotating HA sensor-data display (`ha_sensor_data_0..9`) →
  normal time/date display. Brightness is computed once per frame via an
  adaptive day/night curve (dimmer 18:00–06:00) before any of the branches
  render, so every branch can just call `it.intensity(id(adaptive_brightness_cached))`.
- **Physical button** (`binary_sensor: - platform: gpio`, `button_pin`) —
  five different `on_multi_click` gestures (single/double/hold/long-hold)
  toggle 12h/24h mode, alternate date format, alternate timezone, and text
  replacement/localization modes. See the `binary_sensor:` block before
  adding a sixth gesture — timing windows are already tightly packed.
  # Home Assistant integration
- **`sensor.vfd_clock_data`** — a single HA entity whose *attributes*
  (0 through 9) get read as separate `homeassistant` text_sensors and
  rotated through on-screen. This is how arbitrary HA data (weather, etc.)
  gets shown on this device — no native encrypted API needed for that path.
- **`api: services: - service: message`** — a callable HA service that
  pushes a short-lived message onto the display (`message_alive_time` /
  `message_display_time` / `message_clock_time` control how it interacts
  with the normal clock rotation).
- Text replacement filters (`text_sensor: - platform: template`, `time_text`/
  `date_text`/`dateA_text`) substitute in Chinese characters for AM/PM,
  months, and weekdays when replacement mode is active — see the
  `substitute:` filter lists and the `custom:` glyph-bitmap substitution in
  `substitutions:` if adding another language/character.

## Secrets

Shared `secrets.yaml` with `esphomeclock` (gitignored, not tracked here) —
`wifi1_ssid`/`wifi1_pass` through `wifi3_ssid`/`wifi3_pass`, `esp_fallbackpass`
(used here, see Build/deploy above), `ota_password`. `encryption_key` and
`api_encryption_key` are not referenced by this device's config.
