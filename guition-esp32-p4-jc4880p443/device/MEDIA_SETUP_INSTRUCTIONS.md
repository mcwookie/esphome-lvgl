# Music Assistant control page — setup instructions

Base hardware config: https://github.com/jtenniswood/esphome-lvgl (folder
`guition-esp32-p4-jc4880p443/`). This replaces the original demo page with a
single Music Assistant control page for two players (Roku Express 4K Office,
Brian's Mac mini): play/pause/skip, volume, and 6 playlist shortcuts.

## Files in this drop

- `device/media.yaml` — the Music Assistant page + all sensors/logic (replace)
- `device/lvgl.yaml` — stripped down: core LVGL setup + clock only, no more
  `main_page` (replace)
- `device/device.yaml` — added `rotation: 90` for landscape (replace)
- `package.yaml` — dropped the `sensors:` include entirely; added `media:`
  (replace)

`device/sensors.yaml` is no longer referenced anywhere — you can delete it
or just leave it unused in your fork, either is fine.

## 1. Fork the base repo

Fork https://github.com/jtenniswood/esphome-lvgl to your own GitHub account
if you haven't already — `mcwookie/esphome-lvgl` per our last message.

## 2. Replace the four files above

Copy `device/media.yaml`, `device/lvgl.yaml`, `device/device.yaml`, and
`package.yaml` into the matching paths in your fork, overwriting what's
there now.

**Edit the `ha_url` substitution** at the top of `device/media.yaml` to your
actual LAN-reachable Home Assistant address (e.g. `http://192.168.1.10:8123`)
— not `127.0.0.1`. Used for fetching cover art.

## 3. Add the media icons to assets/icons.yaml

**Required, not optional** — the `glyphs: [...]` list is what tells ESPHome
which characters to compile bitmaps for into `icon_font`; a codepoint not
listed there renders as a blank box regardless of how it's referenced.

Add all 7 of these lines to the existing `glyphs: [...]` list in
`guition-esp32-p4-jc4880p443/assets/icons.yaml`:

```yaml
      "\U000F040A",  # mdi-play
      "\U000F03E4",  # mdi-pause
      "\U000F04AD",  # mdi-skip-next
      "\U000F04AE",  # mdi-skip-previous
      "\U000F0502",  # mdi-television
      "\U000F0322",  # mdi-laptop
      "\U000F04E1",  # mdi-swap-horizontal (no longer used now that there's
                     # only one page, but harmless to leave in if already added)
```

You do **not** need to touch the `substitutions:` section further down that
file — `media.yaml` references icons via raw `\U000F...` escapes directly.

## 4. Point your device config at your fork

Your top-level device YAML (the one in the ESPHome dashboard) should already
point `packages.jtenniswood.esphome_lvgl.url` at
`https://github.com/mcwookie/esphome-lvgl` with
`file: guition-esp32-p4-jc4880p443/package.yaml` — no change needed there
unless you're renaming things.

## 5. Push and recompile

You have `refresh: 0s` set, so it should pick up all four changes without
needing a manual "Clean Build Files" — but if anything looks stale, use it
anyway (ESPHome dashboard → device **⋮ menu → Clean Build Files**).

## What was actually broken (for reference)

Buttons doing nothing were **not** a permission issue (`allow_service_calls`
was already `true` for this device) and **not** a bad Music Assistant ID
(`library://playlist/N` + `media_player.play_media` was verified working by
calling it directly). The real bug: `homeassistant.service`'s `data:` fields
must be **static** — passing `!lambda` directly as a `data:` value (which
every button did, for `entity_id`) silently does nothing. The fix, applied
throughout `media.yaml`, is the documented pattern: compute the lambda once
under `variables:`, then reference it in a Jinja `data_template:` string that
Home Assistant renders on receipt:

```yaml
- homeassistant.action:
    action: media_player.media_play_pause
    variables:
      target_player: !lambda 'return id(current_player);'
    data_template:
      entity_id: "{{ target_player }}"
```

(Also renamed `homeassistant.service:` to the current `homeassistant.action:`
— `service` still works as a backwards-compatible alias, but `action` is the
current name.)

`refresh_display`'s own lvgl/online_image updates were left as plain
`!lambda` — those are pure ESPHome-internal C++ calls with no Home Assistant
templating involved, so inline lambda is correct and was never the problem.

## Known caveats / things to verify after flashing

1. **Landscape orientation direction** — `rotation: 90` was a guess; if the
   image is upside-down for how you've physically mounted the panel, change
   it to `270` in `device/device.yaml`.
2. **Cover art format** — `online_image` is set to `format: JPEG`. If artwork
   doesn't render for some source, try `format: PNG`.
3. **WiFi stability (ESP32-P4 + ESP32-C6)** — known rough edge in the
   ESPHome community for this chip pairing; check the Home Assistant
   Community thread "Guition ESP32 P4 JC4880P443 working config" if the
   panel drops offline intermittently.
4. **Only two players.** To add a third, duplicate the 5-sensor block
   pattern (4 text_sensors + 1 sensor) and add a third player-select button,
   following the Roku/Mac mini pattern in `device/media.yaml`.
