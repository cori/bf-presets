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
| `presets/4.5/vtx/whoop_vtx_g473_fablefpv.txt` | VTX | 4.4, 4.5, 2025.12, 2026.6 | TX power switch on AUX7 — **BETAFPV G473** VTX (25/100/200/400/PIT) |
| `presets/4.5/vtx/whoop_vtx_3level_fablefpv.txt` | VTX | 4.4, 4.5, 2025.12, 2026.6 | TX power switch on AUX7 — **three-level** VTX (25/100/MAX) |
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

Two VTX presets, split by **VTX table family** — not by quad size. Each has a
**clear** option plus one best-effort layout per tier count.

| Preset | For | Options |
|---|---|---|
| `whoop_vtx_g473_fablefpv.txt` | BETAFPV G473 — Air65, Whoozat, Vi, Air75 III | clear · 3 tiers · 5 tiers |
| `whoop_vtx_3level_fablefpv.txt` | three-level VTXs — Hummingbird, HummingRoach, AcroB, BardoII, TINYHAWK_II, NANOHAWK | clear · 3 tiers |

**`clear` writes the idle form of a row** — `vtx N 0 0 0 0 900 900` — which is
what the firmware stores for an unused slot. A zero-width range never matches
and a power of 0 means "leave the power as it is", so the cleared rows cannot
touch the TX power. It leaves `vtx_band` and `vtx_channel` alone, and it sets
`vtx_power = 1` — the table's first entry — because a config left at 0 would
never push a power and the module would sit at its own power-on default.

The **power** in a row is a **one-based index into that quad's own `vtxtable`** —
the firmware looks up `powervalues[index - 1]` and sends that to the VTX module.
`powervalues` is the device value for each entry and `powerlabels` the label for
it. Read your quad's table with `vtxtable powerlevels` / `powervalues` /
`powerlabels`.

| VTX family | `powervalues` | labels | index → level |
|---|---|---|---|
| BETAFPV G473 — Air65, Whoozat, Vi, Air75 III | `1 2 3 4 0` | 25 / 100 / 200 / 400 / PIT | 1=25, 2=100, 3=200, 4=400, 5=PIT |
| Hummingbird, HummingRoach, AcroB, BardoII | `0 1 2` | 25 / 100 / MAX | 1=25, 2=100, 3=MAX |
| TINYHAWK_II, NANOHAWK | `0 1 2` | 25 / 100 / 200 | 1=25, 2=100, 3=200 |
| NANOHAWK_II | `0` | 25 only | 1=25 |
| MOBULA, PROFER | `1 2 14 20 26` | 0 / RCE / 25 / 100 / 400 | 1=0, 2=RCE, 3=25, 4=100, 5=400 |

**Index 0 means "unknown"**, and that is what makes a hand-set power vanish.
`vtxCommonLookupPowerValue()` rejects it, so the FC never pushes a power to
the module at all and the module keeps whatever it powered on at — 200 mW on
the BETAFPV module, MAX on the three-level one. The config is the only source:
`vtxProcessPower()` reads the module's index and, whenever it differs from the
config, pushes the config's index back. So a quad whose `vtx_power` is 0 cannot
be moved by any means, and the FC's own reset default with `USE_VTX_TABLE` is
0 — which is why a quad that was never configured sits at its module's power-on
default.

Two things to remember: `set vtx_power = 1` is the lowest level on every
family, and `set vtx_power` is range checked as 0 … level count − 1, so the
*top* entry of a table cannot be parked with `set` (on a three-level table,
index 3 = MAX is out of range for `set` but fine in a row, because the `vtx`
command checks against the level *count*).

Applying the G473 preset to a three-level quad fails on the 400 mW row with
`invalid argument count`; applying the three-level preset to a G473 quietly
means 25 mW = 100 mW and MAX = 200 mW. Check the table before you apply.

### The boot-time midrc trap

**An active row always overrides a power you set by hand.**
`processRxModes()` calls `vtxUpdateActivatedChannel()` on every RX frame, and
any row whose AUX range is active writes its power into `vtx_power` — the config
value, so it persists. Set 25 mW in the Configurator and it sticks in the UI,
but the FC overwrites it as soon as a row matches.

**And at power-up every channel reads `midrc` (1500)** until the radio links,
so the rows are evaluated straight away against 1500. Any *forced* row whose
range contains 1500 therefore forces its power at **every power-up**, and that
is the value that gets saved — a hand-set power can never survive a reboot.

Two ways out, both used in the presets:

- **Let midrc land on your lowest tier.** Give the lowest tier the range that
  contains 1500: the boot then forces the lowest tier, the safe end of the
  ladder, and a hand-set power is replaced by 25 mW instead of by 200/400 mW.
  This is what the 3-tier options do.
- **Leave a gap at midrc**, or split the tier that covers it into two rows
  either side of 1500. The 5-tier option splits the 200 mW tier into
  1350–1475 and 1525–1600.

Range boundaries are microseconds, but the CLI floors them to 25 µs steps
(`CHANNEL_VALUE_TO_STEP`) and the end of a range is exclusive, so the closest
usable boundaries either side of midrc are **1475** and **1525**.

Note that the switch windows are best-effort — they came from one quad's
captured config, not from a measurement of your radio. On a pot or a switch,
read the actual channel values in the Configurator's channel monitor and adjust
the boundaries to your detents.

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
