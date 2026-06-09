# firestorm-cams-aerosol-data

Speciated aerosol nowcast pipeline. Pulls per-species Aerosol Optical Depth at
550nm from the Copernicus Atmosphere Monitoring Service (CAMS), slims to JSON,
publishes 4×/day to `data/current-aerosol.json` for consumption by DEEPWatch
Global Insights and (eventually) FIRESTORM smoke-vs-dust classification.

## Why this exists

The previous SMOKE layer in DEEPWatch consumed raw PM2.5 from `firestorm-aqi-data`
(Open-Meteo / CAMS bundled mass concentration). That signal cannot distinguish
sea salt from wildfire smoke from Saharan dust — they all register as "PM2.5
≥12 µg/m³". The result was visible artifacts (e.g. apparent smoke streamlines
mid-Pacific at the Roaring Forties, which were actually marine aerosol from
sustained wave-spray over open ocean).

This pipeline pulls CAMS's *speciated* aerosol fields, splitting the signal
into seven physically distinct categories:

| Tag             | Source mechanism                                  | Layer name in DEEPWatch |
|-----------------|---------------------------------------------------|-------------------------|
| sea_salt        | Wave-spray over open ocean                        | MARINE AEROSOL          |
| dust            | Mineral dust from arid land surfaces              | DUST                    |
| organic_matter  | Biomass burning + biogenic SOA                    | SMOKE (with BC)         |
| black_carbon    | Combustion soot                                   | SMOKE (with OM)         |
| sulphate        | Fossil-fuel SO₂ → sulphate + volcanic             | POLLUTION               |
| nitrate         | Vehicle / fertilizer NOₓ → nitrate                | POLLUTION (combined)    |
| ammonium        | Agricultural / livestock NH₃                      | POLLUTION (combined)    |

## Output schema

```jsonc
{
  "updated":         "2026-06-09T11:34:00Z",
  "valid_time":      "2026-06-09T00:00:00Z",
  "resolution_deg":  0.4,
  "species":         ["sea_salt", "dust", "organic_matter", "black_carbon", "sulphate", "nitrate", "ammonium"],
  "stats": {
    "sea_salt":      {"max": 1.23, "mean": 0.04, "p95": 0.18}
    // ...one entry per species
  },
  "grid": [
    [lat, lng, ss, du, om, bc, su, ni, am],
    // ...one row per cell where ANY species exceeds MIN_AOD_THRESHOLD (0.01)
  ],
  "attribution": "Generated using Copernicus Atmosphere Monitoring Service Information 2026."
}
```

`grid` row order is fixed — `[lat, lng, sea_salt, dust, organic_matter, black_carbon, sulphate, nitrate, ammonium]`. Cells where **all seven species are below 0.01 AOD** are dropped (clean ocean, polar gaps).

## Cadence

- Cron 4×/day (UTC 11:00, 13:00, 23:00, 01:00)
- CAMS itself only updates 2×/day (00Z + 12Z runs, ~10h latency)
- Two cron firings per cycle = robustness against late publication or transient ADS errors

## Data source

- **Dataset:** `cams-global-atmospheric-composition-forecasts` on Copernicus ADS
- **Auth:** Personal Access Token via `~/.cdsapirc`. CI: token in GHA secret `CDS_API_TOKEN`, written into `~/.cdsapirc` at job start
- **License:** [CC-BY](https://creativecommons.org/licenses/by/4.0/) — attribution string emitted in every published JSON
- **Citation DOI:** [10.24381/04a0b097](https://doi.org/10.24381/04a0b097)

## Local dev

```bash
echo 'url: https://ads.atmosphere.copernicus.eu/api' > ~/.cdsapirc
echo 'key: <your-PAT>'                              >> ~/.cdsapirc
chmod 600 ~/.cdsapirc

# Mac: brew install eccodes (cfgrib backend)
brew install eccodes

pip install cdsapi xarray cfgrib numpy
python fetch_cams.py
```

## Architecture notes

See [ARCHITECTURE.md](ARCHITECTURE.md) for: why AOD instead of mass mixing
ratios, the run-pick logic, schema rationale, and resilience considerations.
