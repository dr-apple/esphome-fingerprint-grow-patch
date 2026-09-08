# esphome-fingerprint-grow-patch

A patched `fingerprint_grow` component for [ESPHome](https://esphome.io/), plus a full
working project on top of it: a fingerprint-scanner doorbell/entry system on an
ESP32-S3 that talks to Home Assistant over MQTT and unlocks a door strike on a
verified match — inspired by [tinytouch](https://github.com/ZimengXiong/tinyTouch).

- [`components/fingerprint_grow/`](components/fingerprint_grow/) — the patched ESPHome
  component (drop-in replacement for the stock one, via `external_components:`)
- [`esphome/klingel-fingerprint.yaml`](esphome/klingel-fingerprint.yaml) — the full device
  config: enrollment, matching, MQTT events, a security-checked unlock payload, and a
  button-based "wizard" UI (works from Home Assistant *or* a plain browser)
- [`home-assistant/`](home-assistant/) — the HA-side automations, dashboard, and helpers
  that turn the MQTT events into an actual door-unlock system with a management UI

## Why this fork exists

Stock ESPHome's `fingerprint_grow` component has a few rough edges on cheap ZW101-style
Grow-protocol sensor clones (the kind sold on AliExpress/Amazon, not the officially-listed
R307/R503/ZFM-20):

1. **Boot handshake gives up too easily.** Stock sends one password-verify handshake
   ~20ms after boot and permanently marks the sensor as failed if that single attempt
   gets no response. Some clones need longer than that to finish their own power-on
   self-test. This fork keeps `setup()` just as fast/non-blocking as stock (so it never
   eats into WiFi's time-critical early association window), but retries the handshake
   in the background on the component's normal ~500ms polling cadence instead of giving
   up for good.
2. **"Stuck after one scan" during enrollment.** If you use the `sensing_pin` (TouchOut)
   to gate polling — recommended, otherwise ESPHome polls the sensor with a real UART
   image-capture command every 500ms forever, even with no finger present — that pin can
   be unreliable about ever reporting "finger removed" on some clones, silently wedging
   the enrollment/match state machine. This fork adds a 3-second force-timeout: if the
   sensing pin doesn't confirm removal in time, it assumes the finger was removed and
   carries on rather than hanging forever.

See the [commit history](../../commits/main) for the exact diffs and reasoning.

## Hardware

- ESP32-S3 (tested on an ESP32-S3 DevKitC-1 / "Super Mini" board)
- A ZW101-style capacitive fingerprint sensor speaking the common `0xEF01` "Grow"
  packet protocol (same family as the officially-listed R307/R503/R503-RGB/ZFM-20,
  even though only those are documented as tested upstream)
- Wiring: sensor TX/RX → ESP32 UART (GPIO43/GPIO44 in the example config, adjust for
  your board), VCC/GND to a stable 3.3V rail. See tinytouch's own
  [wiring table](https://github.com/ZimengXiong/tinyTouch/blob/main/docs/customer/build.md)
  for the general pin layout if you're using a different sensor module.
- **`sensing_pin` (TouchOut) is deliberately *not* used** in the example config — on the
  unit this was built against it was found to report "finger touching" continuously,
  even seconds after no finger was anywhere near the sensor. tinytouch's own driver
  documents this pin as "not reliable enough" and never gates on it either. Without it,
  ESPHome polls via a real UART command every `update_interval` (500ms default) instead
  — works reliably, at the cost of constant UART traffic.
- **Known hardware limitation:** the Aura LED ring's `aura_led_control`/`led_control`
  commands are acknowledged with `OK` by the sensor (confirmed via DEBUG logs — the
  UART protocol exchange genuinely succeeds), but the physical LED stays permanently on
  "blue breathing" on this unit regardless of the state/color/count sent. This looks like
  a firmware/hardware limitation of this specific cheap sensor clone, not something
  fixable from the ESP32 side — budget for it not working if you're on similar hardware.

## Setup

### 1. Secrets

Add to your ESPHome `secrets.yaml` (alongside your usual `wifi_ssid_*`/`wifi_password_*`,
`mqtt_broker`/`mqtt_username`/`mqtt_password`, `ota_password`):

```yaml
fingerprint_unlock_secret: "<a random token — generate with:
  python3 -c 'import secrets; print(secrets.token_hex(24))'>"
```

This token is embedded in the `finger_matched` MQTT payload and checked by the HA
automation before it will unlock anything — see [Security model](#security-model) below.

### 2. Flash the device

Use [`esphome/klingel-fingerprint.yaml`](esphome/klingel-fingerprint.yaml) as-is, or copy
it into your own ESPHome config and adjust the board/pins/entity names to match your
hardware. It pulls this repo in automatically via `external_components:`.

### 3. Home Assistant

1. Create the 10 helpers listed in [`home-assistant/helpers.md`](home-assistant/helpers.md)
   (one name + one enable-toggle per enrolled finger slot).
2. Import [`home-assistant/automation-tuer-entriegeln.yaml`](home-assistant/automation-tuer-entriegeln.yaml)
   and [`home-assistant/automation-logbuch-ohne-treffer.yaml`](home-assistant/automation-logbuch-ohne-treffer.yaml)
   (paste into an automation's YAML editor, or via the config API). **Replace
   `REPLACE_WITH_YOUR_SECRET`** with the exact value you put in `secrets.yaml`, and
   swap `lock.haustueroeffner` / `script.danny_mobile_push_rich` for your own entities.
3. Add [`home-assistant/dashboard-schliesssystem.yaml`](home-assistant/dashboard-schliesssystem.yaml)
   as a new dashboard view (paste into a view's YAML editor) — gives you per-finger
   enrollment buttons, name/enable toggles, sensor status, and an access-attempt Logbook,
   all in one place. Adjust entity IDs if you renamed anything.
4. Enroll a finger from the dashboard, confirm it shows up correctly in the Logbook, flip
   its "Schließt auf" toggle on, then **only once you trust it**, edit the unlock
   automation's `lock.unlock` action and flip `enabled: false` → not set (it ships
   disabled on purpose, as a safety net for a newly-wired automation controlling a real
   door).

## Security model

The `finger_matched` MQTT payload carries three fields the automation checks, in order,
before it will unlock anything — any failure gets logged to the HA Logbook with the
specific reason instead of silently doing nothing:

1. **Shared secret** — a random token embedded in the payload; without it, anything able
   to publish to the sensor's MQTT topic (e.g. another device sharing the same broker
   login) could otherwise forge a fake match. This is a plaintext payload field, not
   encryption — it stops forged/accidental messages, not a network sniffer capturing and
   replaying a genuine one.
2. **Timestamp (± 2 seconds)** — a Unix timestamp from the ESP32's SNTP-synced clock;
   rejects anything not sent within the last couple of seconds, so even a sniffed and
   later-replayed *genuine* message stops being valid almost immediately.
3. **Per-finger enable toggle** — even a fully valid, fresh, correctly-matched scan is
   rejected if that finger's `input_boolean.fingerabdruck_finger_<N>_aufschliessen_aktiv`
   is off.

**What this does *not* protect against:** a real-time on-path attacker who can both sniff
and immediately re-publish before the 2-second window closes, or anyone with access to
your MQTT broker's credentials for *this* device specifically. For that, add MQTT-over-TLS
and broker-side publish ACLs restricting the `esp_home/fingerprint/#` topic tree to only
this device's own MQTT login — that has to happen on the broker itself, outside what this
repo can configure.

## License

Same as upstream ESPHome (`components/`) — this is a fork/patch, not a rewrite.
