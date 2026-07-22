# ETD Home Range

Computes a subject group's home range using the **Elliptical Time-Density (ETD)** model - a
trajectory-based, nonparametric estimate of an animal's utilization distribution (UD), derived
directly from its own movement behaviour rather than a fitted statistical kernel. The method
builds "time-geography" ellipses between temporally adjacent GPS fixes, sized from a Weibull
distribution fit to the animal's own speed, and sums their overlap across the landscape to produce
a continuous time-density surface (Wall et al. 2014, *Methods in Ecology and Evolution*).

This workflow takes a subject group's fixes all the way from EarthRanger to a percentile-area
table, a percentile choropleth map, and the raw density-surface GeoTIFF - optionally split by
subject, time period, or spatial feature group - and wires the map and table into a dashboard.

## Outputs

Every run writes these artifacts to `$ECOSCOPE_WORKFLOWS_RESULTS`:

| File | Format | Contents |
|---|---|---|
| `[<hash>_]etd_home_range_raster.tif` | GeoTIFF | The raw ETD utilization-distribution surface (continuous density, one band), one per group. |
| `etd_home_range_percentiles.parquet` / `.csv` | GeoParquet + CSV | One row per group per isopleth level: `percentile`, `geometry` (the percentile polygon), `area_sqkm`. |
| `<hash>_etd_home_range_percentiles_<hash>.parquet` / `.csv` | GeoParquet + CSV | Same table, split per group and hash-named to match the dashboard's per-view Files tab. |
| `<hash>_etd_home_range_map.html` | HTML | Percentile choropleth map (colored by isopleth level) over base tile layers, one per group. |
| `<hash>_etd_percentile_table.html` | HTML | Percentile/area table, one per group. |
| `result.json` | JSON | The dashboard: map widget + table widget, one view per group. |

The GeoTIFF's filename is prefixed with the same 6-char group-key hash used everywhere else
(percentile files, dashboard view keys), so a file can always be matched back to the group/view it
belongs to.

The raster and the percentile table are computed from the same trajectory with the same grid
settings (CRS, cell size, nodata, max speed factor, expansion factor) - see the comment in
`spec.yaml` above the `Elliptical Time-Density (ETD)` task-group before changing one without the
other.

## Grouping

Optionally split the trajectory before computing ETD, so each group gets its own map, table, and
GeoTIFF instead of pooling every individual together:

- **Category** - by subject name, subtype, or sex
- **Time** - by any time period (year, month, day, hour, weekday, ...)
- **Spatial** - by EarthRanger spatial feature group (zone)

Left unset, the whole subject group is treated as a single group.

## Pipeline

```
Data Source → Time Range → Subject Group → Group Data (optional groupers)
  → Get Subject Observations (get_subjectgroup_observations)
  → Transform to Relocations (process_relocations)
  → Convert to Trajectory (relocations_to_trajectory)
  → Split Trajectory by Group (split_groups)
  → Calculate ETD Percentiles per group (call_etd_from_combined_params)  → GeoParquet + CSV
  → Generate ETD Raster per group (generate_etd_raster, ext-wd)          → GeoTIFF
  → Color + reproject to WGS84 + draw_map per group                     → Map widget
  → Draw percentile table per group                                    → Table widget
  → Gather Dashboard
```

`call_etd_from_combined_params` is a built-in `ecoscope-platform` task and only returns the
percentile-area polygons - it doesn't expose the underlying raster. `generate_etd_raster` is a
small custom task (in [`ecoscope-workflows-ext-wd`](../wd-partner-tasks/src/ecoscope-workflows-ext-wd))
that calls the same `ecoscope.analysis.UD.calculate_etd_range` function with an `output_path`, so
the raw density surface can be persisted as GeoTIFF too.

The percentile geometry is computed in `EPSG:3857` (meters, for `area_sqkm`) but deck.gl expects
WGS84 lon/lat for its view state, so the map task-group reprojects a copy via `convert_crs` before
drawing - the analysis CRS and the display CRS are kept separate on purpose.

## Setup

`param.yaml` and `test-cases.yaml` are configured for the `mep` EarthRanger data source and subject
group `Elephants`, over 2000-01-01 to 2020-01-01. To point this at a different data source, subject
group, or window, edit:

- `er_client.data_source.name` → your configured EarthRanger data source
- `subject_observations.subject_group_name` → the EarthRanger subject group to compute a home range for
- `time_range` → the analysis window
- `groupers.groupers` → optional list of `index_name` (category), `temporal_index` (time), or
  `spatial_index_name` (spatial feature group) entries

## Build & run

```bash
pixi run compile-etd

cd ecoscope-workflows-etd-workflow
ECOSCOPE_WORKFLOWS_RESULTS="file:///tmp/workflows/etd/output" \
  pixi run ecoscope-workflows-etd-workflow run --config-file ../param.yaml --execution-mode sequential --no-mock-io
```

`./dev/regenerate_rjsf.sh` regenerates just `rjsf.json` from the already-compiled env (no network,
~10s) - useful when iterating on `rjsf-overrides` without a full recompile.
