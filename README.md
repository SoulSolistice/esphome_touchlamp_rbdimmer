[![CI](https://github.com/SoulSolistice/esphome_touchlamp_rbdimmer/actions/workflows/ci.yml/badge.svg)](https://github.com/SoulSolistice/esphome_touchlamp_rbdimmer/actions/workflows/ci.yml)

# Desk Lamp — XIAO ESP32-S3 + DimmerLink (ESPHome)

Retrofit/repair of a touch desk lamp whose original touch dimmer died. The
lamp's **metal base** is reused as the touch electrode, read by the ESP32-S3's
native capacitive touch.

- **Touch the base** → 4 brightness steps (25 / 50 / 75 / 100 %), then the next
  touch turns it **OFF**.
- **Home Assistant** → normal continuous 0–100 % dimming.

Both paths drive the same light, and the touch logic reads the light's *target*
brightness, so a touch after HA has set an arbitrary level jumps to the next
preset above the current brightness.

## Files

```
example/
  desk-lamp.yaml               # entry: identity, board, secrets, includes
  secrets.yaml.example         # copy to secrets.yaml and fill in
packages/
  infrastructure.yaml          # logging, web server, safe mode, status LED, diagnostics
  dimmer.yaml                  # I2C bus + DimmerLink light + sensors + curve select + buttons
  touch.yaml                   # native esp32_touch on the metal base + 4-step cycle
```

## Why this hardware combination

- **DimmerLink (integrated dimmer), not the plain GPIO module.** A dedicated
  Cortex-M on the DimmerLink board does zero-cross detection and TRIAC firing;
  the ESP just writes a brightness value over I²C. All timing-critical work is
  off the ESP, so dimming is flicker-free regardless of Wi-Fi activity, and the
  component is framework-agnostic (it runs fine on ESP-IDF).
- **XIAO ESP32-S3, not the C6.** The S3 has a hardware capacitive-touch
  peripheral; the C6 does not. That's what lets the lamp's metal base be read
  directly, with a software-tunable threshold — exactly what a large electrode
  needs.

## Bill of materials

| Part | Notes |
|------|-------|
| Seeed Studio XIAO ESP32-S3 | native touch on GPIO1–14 |
| DimmerLink-integrated AC dimmer module | I²C @ 0x50; controller + TRIAC on one board |
| ~1 kΩ resistor | in series between the touch GPIO and the metal base |
| The lamp's metal base | the touch electrode (re-used) |

## Wiring

**XIAO ESP32-S3 → DimmerLink (4 logic wires; the dimmer keeps mains on its own
isolated side — never connect the XIAO to mains):**

| XIAO pin | → | DimmerLink | Purpose |
|----------|---|------------|---------|
| `3V3` | → | `VCC` | logic supply (check the board accepts 3.3 V logic) |
| `GND` | → | `GND` | common ground |
| `D4` (GPIO5) | ↔ | `SDA` | I²C data |
| `D5` (GPIO6) | ↔ | `SCL` | I²C clock |

The lamp's mains load wires go to the DimmerLink's screw terminals per its own
markings.

**Touch electrode (the metal base):**

```
metal base ──[ ~1 kΩ ]── D0 (GPIO1)
```

- The series resistor protects the GPIO (ESD / inrush). Keep the wire short.
- **Isolation matters:** the base must float at the ESP's reference — it must
  not be bonded to mains live, neutral, or earth. The dimmer keeps mains on its
  isolated side; the base connects only to the touch GPIO. (This is how the
  original touch lamp was built.)
- The base is a large electrode, so the touch threshold must be calibrated
  (next section).

## Touch calibration (do this once)

The S3 detects touch as a *delta* above a rolling benchmark, and that delta
depends on your specific base, so the shipped `touch_threshold` is only a
placeholder.

1. In `packages/touch.yaml`, set `setup_mode: true`, then flash.
2. Open logs: `esphome logs desk-lamp.yaml` (USB, or OTA once it's online).
3. Watch the per-channel lines:
   ```
   Touch Pad 'Lamp Touch' (Ch1): value=..., benchmark=..., difference=N (set threshold < N to detect touch)
   ```
4. Note `difference` at rest (small, possibly noisy) versus while touching the
   base (a larger positive number). Set `touch_threshold` in `desk-lamp.yaml`
   to roughly halfway between them — comfortably above the resting noise and
   below the touched value.
5. Set `setup_mode: false` and reflash.

If touch is jumpy or unreliable after that, the S3 exposes tuning on the
`esp32_touch:` hub — a debounce/filter group (`debounce_count`, `filter_mode`,
`noise_threshold`, `jitter_step`, `smooth_mode`, which must be set together) and
a denoise pair (`denoise_grade` + `denoise_cap_level`). Reach for those only if
calibration alone isn't stable; the `delayed_on: 50ms` filter already rejects
brief spikes.

## Pin map reference (XIAO ESP32-S3)

Used: `D0`/GPIO1 (touch, channel 1), `D4`/GPIO5 (SDA), `D5`/GPIO6 (SCL). Status
LED is `GPIO21` (active-LOW, onboard yellow LED). `D2`/GPIO3 is a strapping pin
and is left unused; `GPIO0`/`GPIO45`/`GPIO46` are the other strapping pins (not
on the header). Touch-capable header pins are D0/D1/D3/D4/D5/D8/D9/D10
(GPIO1–9), so there's room to relocate the electrode if needed.

## Behaviour detail (touch cycle)

```
OFF ──tap──▶ 25% ──tap──▶ 50% ──tap──▶ 75% ──tap──▶ 100% ──tap──▶ OFF ──▶ …
```

After HA sets, say, 40 %, the next tap goes to 50 % (first preset above
current); from 100 % a tap turns the lamp off. Edit `touch_level_1..4` in
`desk-lamp.yaml` to change the steps.

## A few deliberate choices

- **`restore_mode: ALWAYS_OFF`** on the lamp: it stays off after a power cut
  (predictable, won't surprise-blaze at night) and writes no state to flash.
- **`reboot_timeout: 0s`** on both `wifi` and `api`: a Wi-Fi or Home Assistant
  outage will not reboot a working lamp — touch is fully local. The AP fallback
  is the recovery path. Set `api: reboot_timeout: 15min` if you'd rather have
  automatic recovery than uninterrupted light.
- **Dimming curve** is a runtime **select** entity (DimmerLink handles the
  curve), so you can match it to the bulb without reflashing.
- **ESP-IDF** framework per your guidelines' ESP32 default; DimmerLink and
  esp32_touch both work on it with no special sdkconfig.

## Validate / flash (in your environment)

Schema-validated with `esphome config` against ESPHome 2026.5.3 (valid, no
warnings). The full C++ compile needs the ESP-IDF toolchain, so run it on your
machine:

```bash
cp secrets.yaml.example secrets.yaml      # then fill in real values
esphome config  desk-lamp.yaml            # schema check
esphome compile desk-lamp.yaml            # full build (needs internet 1st time)
esphome run     desk-lamp.yaml            # flash via USB, then OTA after
```

On first boot, check in HA: **Dimmer Ready** is on and **Dimmer Calibrated** is
true, and **AC Frequency** reads ~50/60 Hz — that confirms the DimmerLink is
talking over I²C and has locked onto mains. If the I²C scan finds nothing, the
log's `i2c: scan: true` output will say so (recheck SDA/SCL and power).

## Continuous integration

```markdown
Every pull request and every push to `main` runs the workflow in
`.github/workflows/ci.yml`:

- **Lint** — the same `pre-commit` suite you can run locally (yamllint plus
  trailing-whitespace / end-of-file / line-ending hygiene).
- **Build** — `esphome config` then `esphome compile` against the real
  `example/desk-lamp.yaml`, with a throwaway `secrets.yaml` generated in the job
  (no real secrets are ever committed). This validates the actual packages and
  the exact pinned DimmerLink commit the repo ships, and fails on a schema
  regression, a broken `!include`, or a C++ error in the component.

CI pins ESPHome to a single version (the `ESPHOME_VERSION` env in the workflow);
bump it deliberately. The entry file's `esphome: min_version:` is the matching
runtime floor. The first compile is slow because the ESP-IDF toolchain is a
large download; later runs restore it from cache.

To run the linters locally before pushing:

    pip install pre-commit
    pre-commit install          # optional: run on every git commit
    pre-commit run --all-files
```