
# Sentinel-2 Chlorophyll-*a* (Chl-a) Mapping with Google Earth Engine (Python API + Xee)

This repository contains a reproducible workflow to compute and visualize a chlorophyll-*a* proxy over water bodies from **Sentinel‑2** surface reflectance using the **Google Earth Engine (GEE) Python API**, **geemap**, **xarray**, and **Xee** (xarray–Earth Engine bridge).

> **Author:** Souleymane Mamana Nouri Souley  
> **Notebook:** `GEE_XEE_sen2_chl.ipynb`  
> **Goal:** Build a monthly chlorophyll-*a* (Chl-a) index mosaic (2024–2025) for a user‑drawn Region of Interest (ROI) and export a facet plot as `sen2_chl.png`.

---

## 📦 What this project does
- Links **COPERNICUS/S2_SR_HARMONIZED** (surface reflectance) with **COPERNICUS/S2_CLOUD_PROBABILITY** to mask clouds.
- Derives **NDWI** to restrict analyses to **open water** pixels.
- Computes a simple **Chl‑a index** from Sentinel‑2 bands:  
  \[ **chl** = 4.26 * ( (B3 / B1) ^ 3.94 ) \]
- Converts the Earth Engine ImageCollection to an **xarray.Dataset** via **Xee**.
- **Monthly resamples** (median composites) and generates a **multifacet map** saved to `sen2_chl.png`.

> ⚠️ *The Chl-a expression used here is a generic spectral index (proxy) for demonstration and **is not a site‑calibrated biophysical retrieval**. For scientific/operational use, validate and, if needed, re‑calibrate against in‑situ measurements.*

---

## 🗂️ Data & Collections
- **Sentinel-2 SR (Level‑2A)**: `COPERNICUS/S2_SR_HARMONIZED`
- **S2 Cloud Probability**: `COPERNICUS/S2_CLOUD_PROBABILITY`
- **CRS**: EPSG:4326 when reading to xarray
- **Date range**: 2024‑01‑01 to 2024‑12‑31 (as written: `filterDate('2024','2025')`)

---

## ✅ Requirements
- A Google account **enabled for Earth Engine** (https://earthengine.google.com)
- **Python 3.9+** (recommended)
- Packages:
  - `earthengine-api`
  - `geemap`
  - `xarray`
  - `xee`
  - `matplotlib`

> The notebook also uses the **High Volume** endpoint: `https://earthengine-highvolume.googleapis.com` and a GEE **Cloud Project ID**.

---

## 🛠️ Installation
Create and activate a fresh environment (optional but recommended):

```bash
# with conda
conda create -n gee-chl python=3.11 -y
conda activate gee-chl

# install dependencies
pip install earthengine-api geemap xarray matplotlib xee
```

> If you run the notebook in Jupyter/VS Code, ensure the selected kernel matches the environment where packages are installed.

---

## 🔐 Authenticate & Initialize Earth Engine
In the notebook, the following lines handle authentication and initialization:

```python
import ee

ee.Authenticate()
ee.Initialize(
    project='platinum-pager-426715-c6',
    opt_url='https://earthengine-highvolume.googleapis.com'
)
```

- Replace `project` with **your** GEE Cloud Project ID if different.
- The first run will open a browser for OAuth login. Paste the token back into the notebook when prompted.

---

## 🧭 Usage Workflow (Notebook)
1. **Create a map and draw ROI**
   ```python
   import geemap
   map = geemap.Map(basemap='TERRAIN')
   map  # display the interactive map widget
   
   # Draw a polygon using the toolbar. Then:
   roi = map.draw_last_feature.geometry()
   ```

2. **Define the processing function**
   - Masks **cloudy pixels** using the `probability` band (< 20%).
   - Computes **NDWI** (`(B3 - B8) / (B3 + B8)`) to keep **water** (NDWI > 0.1).
   - Computes the **Chl-a proxy** and keeps image time metadata.

3. **Build ImageCollection**
   ```python
   sen2 = (
       ee.ImageCollection("COPERNICUS/S2_SR_HARMONIZED")
       .linkCollection(ee.ImageCollection("COPERNICUS/S2_CLOUD_PROBABILITY"), 'probability')
       .filterDate('2024', '2025')
       .filterBounds(roi)
       .map(sen2q)
   )
   ```

4. **Load to xarray via Xee**
   ```python
   import xarray as xr
   ds = xr.open_dataset(
       sen2,
       engine='ee',
       crs='EPSG:4326',
       scale=0.001,  # degrees per pixel (~111 m at equator)
       geometry=roi
   )
   ds = ds.sortby('time')
   ```

5. **Monthly composites & plotting**
   ```python
   ds_monthly = ds.resample(time='M').median('time')

   import matplotlib.pyplot as plt
   ds_monthly.chl.plot(
       x='lon', y='lat', col='time', col_wrap=6,
       robust=True, vmin=2, vmax=30, cmap='rainbow', levels=20
   )
   plt.savefig('sen2_chl.png', dpi=360, bbox_inches='tight')
   ```

6. **Output**
   - A multi‑panel figure saved as **`sen2_chl.png`** in the working directory.

---

## 🧪 Method Details
### Cloud Masking
- Uses **S2 Cloud Probability**: pixels with `probability < 20` are retained as *clear*.

### Water Mask
- Computes **NDWI** from S2 bands (`B3`, `B8`) and retains pixels with `NDWI > 0.1` as water.

### Chlorophyll-*a* Proxy
- Expr: `chl = 4.26 * ((B3 / B1) ** 3.94)`
- Bands are scaled by `0.0001` to convert integer reflectance to unit reflectance.
- **Note:** This is a simplified empirical proxy crafted for tutorial purposes; its magnitude is **not** a direct mg·m⁻³ estimate without calibration.

### Temporal Aggregation
- **Monthly median** reduces noise and residual cloud/wave/atmospheric effects.

---

## 🗺️ Tips & Customization
- **Date range:** adjust `.filterDate('YYYY-MM-DD','YYYY-MM-DD')`.
- **Cloud threshold:** increase/decrease `cloud.lt(20)` to balance coverage vs. quality.
- **Water threshold:** tune `ndwi.gt(0.1)` depending on turbidity/adjacency effects.
- **Resolution:** change `scale` in `xr.open_dataset` (in degrees). For ~20 m, use `scale ≈ 0.00018`.
- **Color scaling:** adapt `vmin`, `vmax`, `cmap` to your study area.
- **Export georeferenced rasters:** you can export to GeoTIFF from Earth Engine (e.g., `Export.image.toDrive`) or write arrays from xarray using `rasterio` (not shown in this tutorial).

---

## ⚠️ Known Caveats
- **ROI is required.** If `roi` is `None`, draw a polygon before running processing cells.
- **Xee scales in degrees.** Be mindful of latitude‑dependent ground sampling.
- **Shallow or vegetated waters** can bias NDWI and the Chl‑a proxy.
- **Sunglint / adjacency effects** may need additional corrections in real‑world studies.

---

## 🧯 Troubleshooting
- **`ee.EEException: Please authorize access`**  
  Re‑run `ee.Authenticate()` and ensure you select the Google account that has GEE access, then re‑run `ee.Initialize(...)`.

- **`Earth Engine client not initialized`**  
  Ensure the `ee.Initialize(...)` cell ran successfully. Verify your `project` and `opt_url`.

- **`ModuleNotFoundError: No module named 'xee'`**  
  Run `pip install xee` in the active kernel/environment and restart the kernel.

- **Plot shows blank panels**  
  Check that the ROI intersects valid water and that the date range contains images. Loosen cloud threshold or broaden dates.

---

## 📚 References & Acknowledgments
- **Google Earth Engine** (Python API) and datasets: https://developers.google.com/earth-engine  
- **geemap**: https://geemap.org  
- **Xee** (xarray–Earth Engine engine): https://github.com/gee-community/xee

*This tutorial code is adapted and shared for academic and non‑academic purposes. Please cite data providers and tools as appropriate.*

---

## 📝 License
This tutorial is shared under a permissive, educational intent. If you redistribute or publish results, please acknowledge the author and the underlying data/tool providers.

---

## 🧩 Citation (example)
> Nouri Souley, S. M. (2025). *Sentinel‑2 Water Chlorophyll‑a Index using Python API (Xee)*. Tutorial notebook and code. Retrieved from this repository.



---

## 📊 Example Outputs & Insights
Below are interpretations of the generated chlorophyll-*a* (Chl-a) monthly composite plots for different Regions of Interest (ROIs):

### 1. `sen2_guidimouni_chl.png`
- **ROI**: Small, compact water body (likely a lake or pond).
- **Seasonal pattern**:
  - **Jan–May**: Dominated by red (high Chl-a), suggesting strong algal presence or turbidity.
  - **Jun–Jul**: Almost no data (possible cloud cover or dry season).
  - **Aug–Oct**: Lower values (purple/blue), indicating clearer water or reduced productivity.
  - **Nov–Dec**: Mixed, with high values returning in December.

### 2. `sen2_kokorou_chl.png`
- **ROI**: Elongated shape, possibly a river stretch or floodplain.
- **Coverage**:
  - Sparse in early months (Jan–Jun), likely due to cloud masking or seasonal drying.
  - **Aug–Dec**: More pixels visible, dominated by red (high Chl-a), suggesting seasonal flooding or bloom events.

### 3. `sen2_chl (1).png`
- **ROI**: Curved water body, likely a river bend.
- **Pattern**:
  - **Jan–Jun**: Consistent red along the curve (high Chl-a).
  - **Jul–Aug**: Very few pixels (dry season or heavy cloud cover).
  - **Sep–Dec**: Red returns strongly, indicating bloom resurgence.

### 4. `sen2_lac_chl.png`
- **ROI**: Large, irregular water body (major lake or reservoir).
- **Variability**:
  - **Jan–Jun**: Mixed colors (red + cyan), suggesting spatial variability in Chl-a.
  - **Jul–Aug**: Sparse data (possible seasonal drying or cloud interference).
  - **Sep–Dec**: Red dominates again, indicating widespread bloom or turbidity.

### ✅ Key Observations Across All ROIs
- **Seasonality**: High Chl-a in dry season months (Jan–May, Nov–Dec), low or missing data in wet season (Jun–Aug).
- **Cloud masking and NDWI filtering** reduce coverage during rainy months.
- **Spatial heterogeneity**: Larger water bodies show more color diversity, smaller ones are dominated by single tones.

---
