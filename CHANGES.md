# Changes in this fork

Everything below was found and fixed while actually building and running this project
against real hardware (M5Stack AtomS3 + Atomic QRCode2 Base). Original repo:
[dvejsada/ha-barcode-scanner](https://github.com/dvejsada/ha-barcode-scanner).

## Fixes that apply to both Atom Lite and AtomS3

These are bugs in the shared `esphome/custom_components/uart_readline` component and the
original `.yaml`, unrelated to which board you use:

- `esphome/custom_components/uart_readline/__init__py` was missing its dot — renamed to
  `__init__.py`. Without this, ESPHome doesn't recognize `uart_readline` as a Python
  package at all and the custom component fails to load.
- `text_sensor.py` used `text_sensor.TEXT_SENSOR_SCHEMA`, which no longer exists in
  current ESPHome — replaced with `text_sensor.text_sensor_schema(...)`, the current API.
- `esphome_barcode_scanner.yaml` referenced `${device_name}` for the fallback-hotspot SSID
  but never defined a `substitutions:` block for it — added.

## New: AtomS3 / AtomS3 Lite support

The original `esphome_barcode_scanner.yaml` targets the classic **Atom Lite** (ESP32,
non-native-USB). If you have an **AtomS3 / AtomS3 Lite** (ESP32-S3) instead, use
[`esphome_barcode_scanner_atoms3.yaml`](esphome_barcode_scanner_atoms3.yaml) — the two
boards have different pinouts and this repeatedly needed correcting by testing against the
real device:

| | Atom Lite (original) | AtomS3 (this fork) |
|---|---|---|
| `esp32.board` | `m5stack-atom` | `m5stack-atoms3` |
| UART to scanner (base-bus, TX/RX) | `GPIO19` / `GPIO22` | `GPIO6` / `GPIO5` |
| Button | `GPIO39` | `GPIO41` |
| Status LED | plain GPIO output on `GPIO33` | addressable WS2812 on `GPIO35` |

Two wiring lessons worth calling out for anyone adapting this to yet another board:

1. The Atomic QRCode2 Base is a **"Base" module** — it connects through the bottom
   pogo-pin stacking bus, not the top Grove port. Wiring it to the Grove pins looks like
   it's receiving data (you'll see your own outgoing UART commands reflected back due to
   electrical crosstalk between the unconnected, adjacent pins) but no real reply from the
   module ever arrives.
2. TX/RX orientation isn't documented anywhere obvious for the AtomS3's base bus — it had
   to be found by testing both ways. If you fork this further for other hardware, budget
   for that as a trial-and-error step, not something you can just read off a datasheet.

> **Note for Atom Lite users:** M5Stack's own docs list the Atom Lite's built-in RGB LED as
> an addressable WS2812 on `GPIO27`, not a plain digital output on `GPIO33` as the original
> `esphome_barcode_scanner.yaml` configures. This fork leaves that file untouched because
> we don't have Atom Lite hardware to verify a fix against — but if your LED doesn't work,
> that's the first thing to check.

## New: on/off + manual-trigger button behavior (both YAMLs)

The button previously only did one thing: press it, and the scanner starts a 3-second
decode window. Two problems with that in practice — no debounce (a single physical press
could fire the decode automation ~20 times back to back from switch bounce, which
corrupts the scan cycle so nothing ever gets decoded), and no way to disable scanning.

- **Short press** (50ms–1s): triggers one manual 3-second scan, same as before — but now
  debounced (`delayed_on`/`delayed_off: 25ms` filters) and guarded against overlapping
  triggers so a bouncy press can't launch multiple simultaneous decode sessions.
- **Long press** (1s+): toggles the scanner on/off. The on/off state persists across
  reboots.
- **RGB status LED:**
  - Steady **green** — on, ready to scan
  - Steady **red** — off (short presses are ignored while off)
  - Pulsing **blue** — actively scanning (during the 3s window)
  - Brief **white flash** — a barcode was just successfully decoded (new, valid code)
  - Brief **yellow flash** — valid code, but it's the same one just read last time (repeat)
  - Brief **orange flash** — something was read, but it failed EAN validation (see below)

## New: EAN-5/8/13 validation (AtomS3 YAML)

The scanner occasionally reads partial/corrupted codes (a byte or two dropped over
UART), which previously got treated as real barcode data since the old filter only
checked for a handful of known noise strings and a minimum length of 3 characters.

The `uart_readline` text sensor's filter now validates that every read is a
structurally correct EAN code before accepting it:
- All-digit, and exactly 5, 8, or 13 characters long (EAN-5, EAN-8, EAN-13)
- For EAN-8/EAN-13: the final digit must match the GS1 modulo-10 check digit computed
  from the rest (EAN-5 add-on codes carry no standard check digit, so length is the
  only structural check available for those)

This replaces the old ad-hoc noise-string exclusion list entirely — any protocol echo
or truncated read fails one of these checks and gets dropped, so nothing else needs to
special-case it. The check/flash logic lives together in one filter lambda so there's
never a risk of two conflicting flashes for the same read.

## Data quality note

The scanner module occasionally echoes its own stop-decode command byte back over UART
(`0x75`, i.e. the ASCII character `u`). The EAN validation above rejects this
automatically now (it's neither all-digit nor a valid length), so no separate handling
is needed.
