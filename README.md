# fablefpv's Betaflight presets

Personal Betaflight preset repository. It follows the layout of
[betaflight/firmware-presets](https://github.com/betaflight/firmware-presets) so
it can be loaded directly by the Betaflight Configurator **Presets** tab as a
custom preset source.

## How it's organized

```
presets/<firmware_version>/<category>/<name>.txt
```

- **Firmware version** — the Configurator only shows a preset when the connected
  FC's version `startsWith` one of its `FIRMWARE_VERSION` lines. This is the only
  hard gate; a single file can list several versions.
- **Category** — fixed enum (`OSD`, `RATES`, `MODES`, `TUNE`, `FILTERS`, …).
- **Family / quad / style** — carried by `TITLE` and `KEYWORDS` in the file
  header, which is what the Presets tab's filter dropdowns and search box read.
  Directories are human organization only; they do not affect filtering.

So "whoop vs 3-inch" is a keywords distinction, not a directory one — that's what
you filter on in the UI.

## Current presets

| File | Category | Firmware | Payload |
|---|---|---|---|
| `presets/4.5/rates/whoop_rates_fablefpv.txt` | RATES | 4.4, 4.5, 2025.12, 2026.6 | all four whoop rate profiles |
| `presets/4.5/modes/whoop_modes_fablefpv.txt` | MODES | 4.4, 4.5, 2025.12, 2026.6 | 20 mode slots + 30 adjustment slots |
| `presets/4.5/vtx/whoop_vtx_fablefpv.txt` | VTX | 4.4, 4.5, 2025.12, 2026.6 | VTX band table + AUX-switch control |
| `presets/4.5/osd/whoop_osd_fablefpv.txt` | OSD | 4.4, 4.5 | 115 OSD params |
| `presets/2026.6/osd/whoop_osd_fablefpv.txt` | OSD | 2025.12, 2026.6 | 114 OSD params |
| `presets/4.5/osd/whoop_osd_dvr_fablefpv.txt` | OSD | 4.4, 4.5 | 114 OSD params — flying layout **+ DVR/debug profile 2** |
| `presets/2026.6/osd/whoop_osd_dvr_fablefpv.txt` | OSD | 2025.12, 2026.6 | 113 OSD params — same, ported |

Source of truth: CLI dump `BTFL_cli_WHOOZAT_20261102_225835_BETAFPVG473.txt`
(Betaflight 4.5.0, config rev `93dfc69`) from **Whoozat**, a BetaFPV Air65 65mm
whoop (BETAFPVG473 / STM32G47X). Whoozat is the whoop-family baseline: its OSD
layout, rate profiles, modes and adjustments *are* the whoop presets.

### Why the OSD preset has two variants

The payload is not portable across the whole range, so it is split where the
firmware actually differs. Every `set` line in each file was checked against that
version's `src/main/cli/settings.c`:

- **4.4 / 4.5** differ only by `osd_gps_sats_show_pdop` (absent before 4.5; its
  default is OFF, matching the source quad), so they share one file.
- **2025.12 / 2026.6** — `osd_canvas_width` / `osd_canvas_height` moved behind
  the `OSD_CANVAS_SIZE_DEBUG` build flag and no longer exist on stock builds;
  applying them would raise `Invalid name`. The warnings bitmask is kept as
  saved, but the OSD warning list was reordered and extended after 4.5, so review
  the enabled warnings after applying.
- **Both** omit `osd_nvario_pos`: it only exists on targets with a barometer or
  GPS (the `USE_VARIO` build flag, enabled in `target/common_post.h` only when
  `USE_BARO` or `USE_GPS` is defined). A quad with neither raises
  `Invalid name` on it. Quads that do have a vario simply keep their own value
  for that element.

### The two OSD presets

- **`whoop_osd_*`** — the flying layout, from Whoozat. Profile 2 is a spare
  minimal profile (craft/pilot name, flymode, timer, link quality, battery,
  warnings).
- **`whoop_osd_dvr_*`** — for quads with **no blackbox**, from HummingRoach.
  Profile 1 is the flying layout; **profile 2 is a DVR data profile** that adds
  current draw, mAh drawn, roll/pitch/yaw PIDs, debug, G-force, motor
  diagnostic and ESC RPM — the numbers you would otherwise read out of a
  blackbox log. Select it with `set osd_profile = 2` before the flight so the
  goggle DVR records them, and set it back to 1 afterwards. (There is no mode
  switch for the OSD profile in 4.5; an `ADJUSTMENT_OSD_PROFILE` adjustment is
  the way to get one if you want it in flight.) This preset also omits
  `osd_log_status_pos`, which does not exist on a build without blackbox.

### VTX with AUX control

`whoop_vtx_fablefpv.txt` carries the VTX band table and the AUX-controlled
`vtx` rows captured from Whoozat: AUX1 low / middle / high select band 6
(LOWRACE) channels 1 / 2 / 3, AUX2 at 1750–2000 selects channel 2 and AUX3 at
2000–2100 selects channel 3. It also sets the currently selected VTX (band 4
FATSHARK, channel 8, power level 1) exactly as captured, so applying it will
switch the VTX to that band before the AUX switch is used.

**Rates, modes and adjustments need no splitting** — every parameter, line format
and value was verified against 4.4, 4.5, 2025.12 and 2026.6, so one file each
covers all of them:

- **Rates** switches `rateprofile 0–3` in one pass and leaves profile 0 selected.
- **Modes** writes all 20 mode slots and all 30 adjustment slots, including the
  empty ones, so applying it reproduces the source configuration exactly instead
  of leaving existing unused slots in place.

## Using it in Betaflight Configurator

1. In Configurator, go to **Presets → Preset Sources**.
2. Add a source with:
   - **URL**: `https://github.com/<your-account>/bf-presets`
   - **GitHub branch**: `main`
3. Presets appear filtered by the connected FC's firmware version.

The Configurator reads `index.json` from the root of the branch and fetches each
preset file at its indexed path, so the repository must be pushed to GitHub with
the index in place. It fetches **unauthenticated**, so the repo must be public.

> Applying a preset permanently changes the configuration. Back up first with
> the CLI (`dump all`) and know how to restore it.

## Rebuilding the index

After adding or changing preset files, regenerate the index from the repository
root and commit both files:

```sh
node indexer/indexer.js   # writes index.json and index_hash.txt
node indexer/check.js     # validates the presets (prints OK)
```

The `indexer/` directory is copied verbatim from
betaflight/firmware-presets (GPL-3.0, see `LICENSE`).
