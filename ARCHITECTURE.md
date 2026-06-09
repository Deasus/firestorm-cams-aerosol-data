# Architecture — firestorm-cams-aerosol-data

## Why CAMS, not Open-Meteo / EPA AirNow / commercial

Open-Meteo's air-quality endpoint serves a CAMS-derived PM2.5 mass concentration,
but only as a single bundled total. We need **speciated** output — sea salt vs
dust vs smoke vs sulphate are physically and operationally distinct, and a
visualization layer that conflates them is misleading. CAMS itself publishes
each species separately; that's the only path that gets us what we need.

Other rejected options:

- **EPA AirNow** — US-only, ground-station-based, no global signal, no
  speciation.
- **NASA EOSDIS / FIRMS** — fire detections (heat signatures), not aerosol
  concentrations. Useful as a complement, not a replacement.
- **OpenAQ** — aggregator over ground stations, sparse global coverage, no
  speciation.
- **Sentinel-5P** — measures *column* concentrations (NO₂, SO₂, CO, aerosol
  index) but not the per-species AOD breakdown, and the access pattern is
  heavier (Sentinel Hub OAuth, raster tiles).

## Why AOD at 550nm, not mass mixing ratios

Two alternatives exist for "how much of species X is in the air":

1. **Mass mixing ratio** (`aermr01..aermr11`) — kilograms of aerosol per
   kilogram of air. Defined at every model level (~137 vertical layers).
   To get a surface concentration you must also pull pressure or model level
   data, which multiplies request size by 137 and pushes past ADS limits.

2. **Aerosol Optical Depth at 550nm** (`*aod550nm`) — vertically-integrated
   optical thickness at the visible green wavelength. Single-level, dimensionless,
   bounded ~0–3 in practice.

We pick AOD because:
- Single-level → small payload, fits ADS request limits
- Visually intuitive (high = "thick plume")
- Same canonical product as NASA Worldview, Windy Pro, Earth Nullschool
- Trivially color-mappable for the canvas-particle layer

If a downstream consumer ever needs ground-level concentrations specifically
(for AQI calculation), we'd add a second pipeline `firestorm-cams-surface-mass-data`
that pulls just the surface model level (level 137) for the same species.

## Run-pick logic

CAMS publishes:
- **00Z analysis** — guaranteed available by 10:00 UTC same day
- **12Z analysis** — guaranteed available by 22:00 UTC same day

Our `pick_run_date()` walks backward from wall-clock to the most recent
guaranteed-ready cycle. This avoids 404s from chasing a cycle that hasn't been
published yet and is more robust than a fixed offset.

Cron schedule (4×/day): 11:00, 13:00, 23:00, 01:00 UTC. The dual firings per
cycle act as a transient-error backstop — if one fails (ADS queue blip, runner
hiccup), the second catches it.

## Output size

After dropping cells where all species are below 0.01 AOD threshold:

- 0.4° native grid: 901 × 451 = 406,251 cells, ~30% retained → ~125,000 rows
- Per row: 9 floats × ~6 bytes JSON = ~54 bytes
- Total: ~6.5 MB uncompressed JSON, ~1.5 MB gzip
- raw.githubusercontent serves it gzipped automatically

If size becomes a frontend bottleneck, set `DOWNSAMPLE_FACTOR = 2` in
`fetch_cams.py` (drops to 0.8° resolution, ~1.6 MB uncompressed).

## Resilience notes

- ADS is non-operational by their own admission — outages happen. If `cdsapi`
  errors, the GHA job fails loudly and the previous `current-aerosol.json`
  stays live. No code change needed; next cron run retries.
- CAMS short-names occasionally change with ECMWF cycle upgrades. The
  `short_to_tag` dict in `parse_and_slim()` will print WARN lines if a new
  short-name appears, and the script exits with FATAL if no recognized species
  parse — better to fail loudly than to publish a corrupt JSON.
- License compliance: every published JSON includes the attribution string
  per Copernicus CC-BY policy. Frontend should render it visibly somewhere
  (footer is fine).
