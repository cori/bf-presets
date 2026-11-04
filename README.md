# Whoozat Betaflight presets

Personal Betaflight preset repository for **Whoozat** — a BetaFPV Air65 65mm
whoop (BETAFPVG473 / STM32G47X, "SG47"). It follows the layout of
[betaflight/firmware-presets](https://github.com/betaflight/firmware-presets) so
it can be loaded directly by the Betaflight Configurator **Presets** tab as a
custom preset source.

## Contents

| Preset | Category | Firmware | What it contains |
|---|---|---|---|
| `presets/4.5/osd/whoozat_osd_cori.txt` | OSD | 4.5 | All 117 OSD parameters as saved on the quad |
| `presets/4.5/rates/whoozat_rates_cori.txt` | RATES | 4.5 | All four rate profiles (RaceXpo, JuicyXpo, SbangXpo, SBANG) |
| `presets/2026.6/osd/whoozat_osd_cori.txt` | OSD | 2026.6 | Same OSD settings, ported to 2026.6 |
| `presets/2026.6/rates/whoozat_rates_cori.txt` | RATES | 2026.6 | Same rate profiles, ported to 2026.6 |

Source: CLI dump `BTFL_cli_WHOOZAT_20261102_225835_BETAFPVG473.txt`
(Betaflight 4.5.0, config rev 93dfc69), taken from the quad on 2026-11-02.

### Notes on the 2026.6 ports

- `osd_canvas_width` / `osd_canvas_height` are omitted: in 2026.6 they moved
  behind the `OSD_CANVAS_SIZE_DEBUG` build flag and no longer exist on stock
  builds (applying them would raise `Invalid name`).
- Every other parameter name and value range was verified against the 2026.6.2
  firmware settings table.
- The warnings bitmask is kept as saved, but the OSD warning list was reordered
  and extended since 4.5 — review the enabled warnings after applying.
- Element positions keep the OSD-profile visibility bits (bits 11–13) exactly as
  stored on the FC.

## Using it in Betaflight Configurator

1. In Configurator, go to **Presets → Preset Sources**.
2. Add a source with:
   - **URL**: `https://github.com/<your-account>/bf-presets-whoozat`
   - **GitHub branch**: `main`
3. The presets appear in the Presets tab, filtered by the connected flight
   controller's firmware version (4.5.x or 2026.6.x).

The Configurator reads `index.json` from the root of the chosen branch, so the
repository must be pushed to GitHub with the index in place.

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
