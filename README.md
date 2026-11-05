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
| `presets/4.5/vtx/whoop_vtx_fablefpv.txt` | VTX | 4.4, 4.5, 2025.12, 2026.6 | TX power switch on AUX7 (2/3/5-position options) |
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

### VTX TX-power switch (AUX7)

`whoop_vtx_fablefpv.txt` is a **TX power switch on AUX7**, nothing more. Band
and channel are left at `0`, which the firmware reads as "do not change the
current band or channel", so this preset touches neither the band table, the
power ladder, nor the selected band/channel/power — the VTX manufacturer
settings are left exactly as configured.

**An active row always overrides a power you set by hand.** `processRxModes()`
calls `vtxUpdateActivatedChannel()` on every RX frame, and any row whose AUX
range is active writes its power into `vtx_power` — the config value, so it
persists. Set 25 mW in the Configurator and it sticks in the UI, but the FC
overwrites it as soon as a row matches, and a reconnect re-reads the FC's
value. Park the switch in a detent no row covers if you want a hand-set power
to stick.

Power is a level index into the factory ladder (0 = 25 mW, 1 = 100, 2 = 200,
3 = 400, 4 = PIT). The firmware only applies a row's power when it is **greater
than zero**, so level 0 — the 25 mW level — can never be selected by a row; it
means "leave the power as it is". The options therefore use levels 1–4, and the
5-position option deliberately leaves the lowest detent unmapped so a hand-set
25 mW survives there.

| Option | Ranges | Power |
|---|---|---|
| 2 positions | 900–1500 / 1500–2100 | 100 / 400 mW |
| **3 positions** *(default)* | 900–1150 / 1150–1500 / 1500–2100 | 100 / 200 / 400 mW |
| 5 positions | 900–1150 / 1150–1500 / 1500–1750 / 1750–2000 / 2000–2100 | *manual* / 100 / 200 / 400 / PIT |

The 3-position option is the default because it reproduces the captured
configuration. Note that the original dump's rows 3 and 4 were unreachable —
row 2 covered 1500–2100 and shadowed them — so the default option clears them
rather than preserving a dead pair.

Firmware note: 4.5 stops at the first matching row, while 2026.6 applies every
matching row and lets the highest-indexed one win per field. The rows here are
disjoint, so it makes no difference — but a hand-set power loses either way.

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
