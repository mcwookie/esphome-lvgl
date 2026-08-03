# Music Assistant control page — setup instructions

Base hardware config: https://github.com/jtenniswood/esphome-lvgl (folder
`guition-esp32-p4-jc4880p443/`). This adds a new LVGL page to it for
controlling two Music Assistant players (Roku Express 4K Office, Brian's
Mac mini) via play/pause/skip, volume, and 6 playlist shortcuts.

## 1. Fork the base repo

Fork https://github.com/jtenniswood/esphome-lvgl to your own GitHub account
(or a private repo) — you need to add a file to it, and `esphome.yaml`
pulls the package straight from GitHub, so it needs to point at a copy you
control.

## 2. Add device/media.yaml

Copy the attached `media.yaml` into `guition-esp32-p4-jc4880p443/device/media.yaml`
in your fork.

**Edit the `ha_url` substitution at the top of the file** to your actual
LAN-reachable Home Assistant address (e.g. `http://192.168.1.10:8123`) —
NOT `127.0.0.1`, which only resolves inside HA's own container and won't
be reachable from the panel. This is used to fetch cover art images.

## 3. Wire it into package.yaml

In `guition-esp32-p4-jc4880p443/package.yaml`, add one line:

```yaml
packages:
  device: !include device/device.yaml
  sensors: !include device/sensors.yaml
  lvgl: !include device/lvgl.yaml
  media: !include device/media.yaml        # <-- add this line
  time: !include addon/time.yaml
  backlight: !include addon/backlight.yaml
  network: !include addon/network.yaml
  fonts: !include assets/fonts.yaml
  icons: !include assets/icons.yaml
  button: !include theme/button.yaml
```

## 4. Add the media icons to assets/icons.yaml

**This step is required, not optional.** The `glyphs: [...]` list is what
tells ESPHome which characters to actually compile bitmaps for into
`icon_font` — if a codepoint isn't listed there, referencing it anywhere
(raw escape or named substitution) renders as a blank box, since the font
simply doesn't contain that glyph.

Edit `guition-esp32-p4-jc4880p443/assets/icons.yaml` and add all 7 of these
lines to the existing `glyphs: [...]` list:

```yaml
      "\U000F040A",  # mdi-play
      "\U000F03E4",  # mdi-pause
      "\U000F04AD",  # mdi-skip-next
      "\U000F04AE",  # mdi-skip-previous
      "\U000F0502",  # mdi-television
      "\U000F0322",  # mdi-laptop
      "\U000F04E1",  # mdi-swap-horizontal
```

You do **not** need to add anything to the `substitutions:` section further
down that same file — that block just maps friendly names (like `$lightbulb`)
to codepoints for readability in `device/lvgl.yaml`. `device/media.yaml`
already references these icons via their raw `\U000F...` escapes directly,
so no substitution names are needed for this page to work.

## 5. Point your device config at your fork

In your device's `esphome.yaml` (the file you paste into the ESPHome
dashboard/CLI config), change the `packages.setup.url` to your fork's URL
instead of `jtenniswood/esphome-lvgl`, and set your `name`/`friendly_name`/
`room` substitutions and `wifi_ssid`/`wifi_password` secrets as usual.

## 6. Flash and test

- First flash must be over USB (ESP32-P4 + ESP32-C6 boards can be finicky
  over WiFi for the very first connection).
- After boot, you should see the existing `main_page` (if you kept the
  original scene/blind buttons — delete those from `device/lvgl.yaml` if
  you don't want them) plus a small swap icon (⇄) in the top-right corner
  that switches to the new `music_page`.
- Tap "Office Roku" or "Mac mini" to pick a target, then use play/pause,
  skip, volume, or a playlist button.

## Known caveats / things to verify after flashing

1. **`media_player.play_media` ID format** — the playlist buttons pass
   Music Assistant's internal `library://playlist/N` IDs directly as
   `media_content_id`. This is the same approach used by the Home
   Assistant automation already set up for the Roku via the
   `input_select.office_roku_playlist_selector` helper, and matches what
   `media_player.play_media` expects for Music Assistant targets. If a
   playlist button doesn't play anything, check Home Assistant's logs for
   the `media_player.play_media` service call error.
2. **Cover art format** — `online_image` is configured for `format: JPEG`.
   If artwork doesn't render for some player/source, the image may be PNG
   — try changing `format: JPEG` to `format: PNG` in `device/media.yaml`.
3. **WiFi stability (ESP32-P4 + ESP32-C6)** — this chip pairing has some
   reported WiFi reconnect issues in the ESPHome community. If the panel
   drops offline intermittently, check the Home Assistant Community
   thread "Guition ESP32 P4 JC4880P443 working config" for the latest
   community fixes.
4. **This only controls two players.** To add a third (e.g. BRAVIA 7 or
   Roku Ultra), duplicate the sensor block pattern (4 text_sensors + 1
   sensor per player) and add a third player-select button, following the
   same pattern as the Roku/Mac mini blocks in `device/media.yaml`.
