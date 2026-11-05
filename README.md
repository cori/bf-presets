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

`whoop_vtx_g473_fablefpv.txt` is a **TX power switch on AUX7**, nothing more. Band
and channel are left at `0`, which the firmware reads as "do not change the
current band or channel", so this preset touches neither the band table, the
power ladder, nor the selected band/channel — the VTX manufacturer settings are
left exactly as configured.

**An active row always overrides a power you set by hand.** `processRxModes()`
calls `vtxUpdateActivatedChannel()` on every RX frame, and any row whose AUX
range is active writes its power into `vtx_power` — the config value, so it
persists. Set 25 mW in the Configurator and it sticks in the UI, but the FC
overwrites it as soon as a row matches.

**And at power-up every channel reads `midrc` (1500)** until the radio links,
so the rows are evaluated straight away against 1500. Any *forced* row whose
range contains 1500 therefore forces its power at **every power-up**, and that
is the value that gets saved — a hand-set power can never survive a reboot.
The constraint is only about the row that *contains* 1500: levels may well be
forced on either side of it, so the ladder can be split around midrc.

The **power index** is an index into *that quad's own* VTX table — it is **not**
a universal power level. The CLI validates it against the table's level count,
so a power above it is rejected with `invalid argument count` and the whole row
is cleared. Read your quad's table with:

```
vtxtable powerlevels
vtxtable powervalues
vtxtable powerlabels
```

| VTX family | `powervalues` | labels | usable in a row |
|---|---|---|---|
| BETAFPV G473 — Air65, Whoozat, Vi | `1 2 3 4 0` | 25 / 100 / 200 / 400 / PIT | 1, 2, 3, 4 |
| Hummingbird, HummingRoach, AcroB, BardoII | `0 1 2` | 25 / 100 / MAX | 1, 2 |
| TINYHAWK_II, NANOHAWK | `0 1 2` | 25 / 100 / 200 | 1, 2 |
| NANOHAWK_II | `0` | 25 | — (only the parked value) |
| MOBULA, PROFER | `1 2 14 20 26` | 0 / RCE / 25 / 100 / 400 | 14, 20, 26 |

So there are two VTX presets, and **pick the one that matches your quad's
table**:

- **`whoop_vtx_g473_fablefpv.txt`** — BETAFPV G473 (25 / 100 / 200 / 400 / PIT).
  On that ladder index 1 = 25 mW, 2 = 100, 3 = 200, 4 = 400 (MAX), 0 = PIT.
- **`whoop_vtx_3level_fablefpv.txt`** — three-level ladders (25 / 100 / MAX).
  Here index **0 = 25 mW**, which is also the firmware's "leave as set"
  sentinel, so 25 mW can never be forced by a row — it is the parked value
  (`set vtx_power = 0`), and only indices 1 and 2 can force a level. Its switch
  windows come from the HummingRoach (900–1350 / 1350–1700 / 1700–2100).

The HummingRoach's own captured configuration was written **one index too
high**: it used powers 1, 2 and 3 on a ladder where 0 is 25 mW, 1 is 100 mW and
2 is MAX. That meant it ran a step hot (100 mW where 25 mW was intended) and
its top row pointed at an index the device does not have. The three-level preset
fixes that.

Applying the G473 preset to a three-level quad fails on the 400 mW row with
`invalid argument count`; applying the three-level preset to a G473 quietly
means 25 mW = 100 mW and MAX = 200 mW. Check the table before you apply.

A row with power `0` means "leave the power as it is" — the firmware only
applies a row power when it is `> 0` — so **PIT cannot be selected by a row at
all**; set it by hand in the Video Transmitter tab. Note the trap: `set vtx_power = 0`
is therefore *PIT mode*, not 25 mW. 25 mW is index 1.

Range boundaries are microseconds, but the CLI floors them to 25 µs steps
(`CHANNEL_VALUE_TO_STEP`) and the end of a range is exclusive, so the closest
usable boundaries either side of midrc are **1475** and **1525** — 1499 lands on
1475 and 1501 lands on 1500.

The 3-tier options are the hummingbird layout — 25 mW, 100 mW, MAX — using the
switch windows from the mode ranges in the same dump (900–1300, 1300–1675,
1675–2100). The mid tier cannot simply be 1300–1675: that range contains 1500
and would fire at every power-up.

| Option | Forced rows | Notes |
|---|---|---|
| **3 tiers: 25 / 100 / MAX** *(default)* | 25 @ 900–1150, 100 @ 1150–1475, MAX @ 1525–2100 | mid tier cut short at 1475, gap 1475–1525 |
| 3 tiers: 25 / 100 / MAX (mid tier split around 1500) | 25 @ 900–1150, 100 @ 1150–1475, 100 @ 1525–1700, MAX @ 1700–2100 | mid tier either side of 1500, gap 1475–1525 |
| 3 tiers: 25 / 100 / MAX (mid tier covers 1500) | 25 @ 900–1150, 100 @ 1150–1525, MAX @ 1525–2100 | **not boot safe** — 100 mW at every power-up |
| 3 tiers: 25 / 100 / 200 mW | 25 @ 900–1150, 100 @ 1150–1475, 200 @ 1525–2100 | gentler top for a whoop frame |
| 5 positions: 25 / 100 / 25 / 200 / MAX | 25 @ 900–1150, 100 @ 1150–1475, 200 @ 1525–1850, MAX @ 1850–2100 | 1475–1525 unmapped, holds the hand-set power |
| manual (25 mW) / 200 / MAX mW | 200 @ 1525–1800, MAX @ 1800–2100 | one wide unmapped region, 900–1525 |
| as captured *(not boot safe)* | 25 @ 900–1150, 100 @ 1150–1500, 200 @ 1500–2100, 100 @ 1750–2000, 200 @ 2000–2100 | the 1500–2100 row fires at every power-up |

**Which 3-tier option is right depends on where your radio sends the middle
switch position.** If it reads below 1475, the first option gives it 100 mW. If
it reads exactly 1500, it lands in that option's gap and keeps the last power —
use the split option instead. If you would rather the middle position always
takes 100 mW and don't mind the VTX coming up at 100 mW, use the option that
covers 1500. Watch the AUX7 value in the Configurator's channel monitor to
settle it.

Two version differences worth knowing: 4.5 stops at the first matching row,
while 2026.6 applies every matching row and lets the highest-indexed one win
per field. And "leave as set" means *leave whatever was last set*: visit a
higher detent and the lower one keeps that value until you re-pick it in the
Video Transmitter tab.

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
