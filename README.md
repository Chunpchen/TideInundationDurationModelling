# Tidal inundation duration along the Chinese coast

This notebook estimates how many hours per day a tidal-flat pixel is under water. It compares hourly astronomical tides from TPXO10-atlas-v2 with a DEM adjusted for sediment accretion, land subsidence and sea-level rise. Each run writes one raster per tile and target year. The target year controls the elevation adjustment. The same 2022–2023 tidal forcing is used in each projection.

## Files and inputs

| Item | Contents |
| --- | --- |
| `Tide_inundation_duration.ipynb` | Run settings, model functions, decadal sea-level values, 86 tile-specific accretion rates and the tile loop |
| `requirements.txt` | Versions of the Python packages used to run the example |
| `data/dem/{id}_dem.tif` | 86 model-input DEM tiles, numbered 1–86. These resampled rasters contain elevations in metres. Raster spacing is encoded in each GeoTIFF. Tile 6, for example, has a spacing of approximately 0.000897° on both axes. The original high-resolution DEM is a separate study product. |
| `tide_models/TPXO10_atlas_v2/` | External tide-model files to install separately. Use the TPXO10-atlas-v2 **NetCDF** distribution, including the grid and elevation harmonic constituents. |

TPXO heights are relative to mean sea level. Supply DEM elevations on a compatible vertical reference. The coordinate transformation in the notebook converts horizontal positions to WGS 84. It applies no vertical-datum offset. Obtain the model through the provider's registration process and retain the directory structure expected by eo-tides. [TPXO10-atlas access](https://www.tpxo.net/global/tpxo10-atlas) · [eo-tides model setup](https://geoscienceaustralia.github.io/eo-tides/setup/)

## Install and run the example

The example was executed on Windows with Python 3.13.5. From the directory containing this README, create an environment and select it as the notebook kernel.

```sh
conda create -n tidal-inundation python=3.13.5 pip
conda activate tidal-inundation
python -m pip install -r requirements.txt
python -m ipykernel install --user --name tidal-inundation --display-name "Python (tidal-inundation)"
python -m jupyterlab Tide_inundation_duration.ipynb
```

1. Open the notebook and select **Python (tidal-inundation)** as the kernel.
2. In the first code cell, set `TIDE_MODEL_DIR` to the parent of `TPXO10_atlas_v2`. For example, if the grid file is `/data/tides/TPXO10_atlas_v2/grid_tpxo10atlas_v2.nc`, set `TIDE_MODEL_DIR = "/data/tides"`. Launch Jupyter from the package directory for the other relative paths to work.
3. Keep `GRID_IDS = [6]` and select **Run All Cells**. The first run computes hourly tides, writes the tide cache and calculates inundation. A second run reuses the cache.
4. Open `results/SSP2-4.5_k0.05_S0.002/2040/avg_daily_flood_6_2040.tif`. The test produced a 33-row by 23-column raster, with 160 valid pixels, a range of 0.0–24.0 h/day and a mean of 18.11125 h/day across valid pixels after rounding each pixel to one decimal place. The first computation took about one minute on the test machine.

The dependencies include NumPy, pandas and SciPy for arrays, time series, interpolation and statistics. Rasterio and pyproj handle rasters and geographic coordinates. scikit-learn performs the nearest-neighbour grid search. eo-tides uses pyTMD for tide prediction. xarray and netCDF4 store hourly tides. GeoPandas, Shapely and pyogrio write point features. Matplotlib draws the coverage map, and tqdm reports progress.

## Settings for other runs

| Setting | Default | Interpretation |
| --- | --- | --- |
| `GRID_IDS` | `[6]` | Set to `range(1, 87)` for all tiles. Tile *i* uses entry *i − 1* of the Python list `A`. |
| `START_DATE`, `END_DATE` | 1 Jan 2022 00:00 to 31 Dec 2023 23:00 | Hourly UTC timestamps, inclusive. Use whole days beginning at midnight. The default has 17,520 hours in 730 days. |
| `GRID_RESOLUTION`, `BUFFER_DISTANCE` | 0.03°, 0.03° | Target grid spacing and maximum angular distance from a valid DEM pixel. Bounds are padded by the same buffer. |
| `INTERPOLATION_METHOD` | `nearest` | Spatial interpolation of predicted tide height from the grid to each DEM location. With `linear`, all contributing neighbours must have usable predictions. |
| `BASE_YEAR`, `YEAR` | 2023, 2040 | Use 2023 for a baseline run or a later decade in the embedded table (2030–2150) for a projection. |
| `SSP`, `k`, `S` | SSP2-4.5, 0.05, 0.002 m/year | Sea-level scenario, annual fractional decline in accretion rate, and annual subsidence. The example retains the parameter combination in the public notebook. |

For the three paired projections in the study, set `(SSP, k)` to `("SSP1-2.6", 0.01)`, `("SSP2-4.5", 0.03)` or `("SSP5-8.5", 0.05)`. Select the target year and rerun the cells. The historical tide series can be reused across these scenarios. Use a fresh cache directory if you replace a DEM in place, change the tide-model data or change the model software. Choose a new output root when comparing different tide windows or interpolators.

## Calculation

1. Read the first DEM band and its NoData mask as defined by the NoData value and NaNs. Transform the raster's integer column and row positions to longitude and latitude using its affine transform and CRS. Construct a geographic prediction grid and retain points within the specified buffer of valid DEM positions.
2. Predict hourly tide height in metres at each retained grid point, then interpolate each hourly field to the DEM positions.
3. Let `T = YEAR − 2023` and `A_i` be the initial accretion rate for tile *i*, in cm/year. Cumulative accretion in metres is `0.01 × A_i × sum((1 − k)^t for t = 1,…,T)`. The first projected year uses exponent 1. For projections, the sea-level increment equals the selected tabulated level minus the fixed 0.049 m reference used in the original code. The baseline run sets the increment to zero.
4. Compute adjusted elevation as `DEM + cumulative accretion − S × T − sea-level increment`. Count an hour as flooded when its tide height exceeds adjusted elevation. For each valid pixel, sum the flooded hours in each complete 24-hour day and take the arithmetic mean over days. Save the result to one decimal place.

## Outputs and checks

| Output | Interpretation |
| --- | --- |
| `avg_daily_flood_{id}_{year}.tif` | Mean daily inundation in h/day, from 0 to 24, on the input raster grid. Float32 with NoData = −9999. |
| `grid_coverage_{id}.png` | DEM footprint, candidate grid points and retained prediction points when `PLOT_GRID_COVERAGE = True`. |
| `tide_grid_points_{id}.shp` and sidecar files | With `TIDE_GRID_FORMAT = "shp"`, pointwise minimum, maximum, mean and standard deviation of tide height in metres. Keep the sidecar files together. |
| `tide_grid_{id}.nc` | With `TIDE_GRID_FORMAT = "netcdf"`, hourly tide heights at retained points. |
| `cache/tide_cache_{id}.nc` | Hourly tide array, coordinates, valid grid mask and run settings used by later runs. |

The diagnostic map and optional tide-grid export are created when tides are calculated. Set `USE_TIDE_CACHE = False` to regenerate them. Check the coverage map if interpolation reports missing tide heights. The notebook keeps an hourly Boolean mask in memory, requiring approximately `number of hours × rows × columns` bytes for this array alone. Process tiles sequentially.

## Scope of verification

Using the same tile and tide series, the original GitHub notebook and this English version gave identical values for all 160 valid pixels in the example. Freshly predicted tides also matched the archived tile-6 tide cache. This verifies the published code path for one tile. A separate historical inundation raster for the same tile and scenario has a mean of 22.28937 h/day. Subtracting an additional 1.0 m from tile-6 elevation reproduces that raster's 160 values. The provenance and intended use of this additional offset require confirmation before the example can be identified with the manuscript production run.

The notebook produces the inundation-duration rasters. DEM construction, habitat thresholding, national mosaicking, figure statistics and publication graphics are separate analyses. [Original GitHub notebook](https://github.com/Chunpchen/TideInundationDurationModelling/blob/1fd8e8ae192bd33fce4a420a80433d76d9c89a3f/Tide_inundation_duration.ipynb)
