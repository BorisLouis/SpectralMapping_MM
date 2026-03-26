# Spectral Map Reconstruction — MicroManager OME-TIF

A Google Colab notebook for reconstructing 2D hyperspectral maps from MicroManager OME-TIF acquisitions. Reads a full XY stage scan, averages each image along the spatial axis to improve signal-to-noise ratio, and builds an interactive spectral data cube you can explore pixel by pixel.

---

## What it does

MicroManager saves one `.ome.tif` file per stage position during a 2D XY scan. Each image is elongated along X (wavelength axis) and has multiple rows along Y (spatial rows). This notebook:

1. Reads all metadata from a single file (stage positions, grid geometry, image dimensions)
2. Builds a `(rows × cols × wavelengths)` spectral cube by averaging each image along Y
3. Plots a 2D intensity map with physical µm axes
4. Provides an interactive spectrum viewer (slider-based, one pixel at a time)
5. Supports optional wavelength calibration (pixel → nm)
6. Supports band-integrated maps over any wavelength range
7. Saves the spectral cube as a `.npz` archive to Google Drive

---

## Requirements

- Google account with Google Drive
- Google Colab (free tier is sufficient)
- Data acquired with MicroManager using the MMStack format (`.ome.tif`)
- Python packages: `tifffile`, `numpy`, `matplotlib`, `tqdm` (all installed automatically in the notebook)

---

## Data format expected

The notebook expects a folder on Google Drive containing files named:

```
<PREFIX>_MMStack_Posvoid.ome.tif
<PREFIX>_MMStack_Posvoid1.ome.tif
<PREFIX>_MMStack_Posvoid2.ome.tif
...
<PREFIX>_MMStack_PosvoidN.ome.tif
```

Where `PREFIX` matches the name of the containing folder (MicroManager's default naming convention). The first file (no number suffix) contains the full acquisition metadata for all positions, as well as the complete image stack.

Each image should be:
- `height × width` pixels (e.g. `30 × 640`)
- Y axis = spatial rows (will be averaged for SNR improvement)
- X axis = wavelength / spectral axis

---

## How to use

### Step 1 — Upload your data to Google Drive

Put your scan folder somewhere on Google Drive, for example:

```
My Drive/
  microscopy_data/
    my-scan_1/
      my-scan_1_MMStack_Posvoid.ome.tif
      my-scan_1_MMStack_Posvoid1.ome.tif
      ...
```

### Step 2 — Open the notebook in Google Colab

Upload `spectral_map_reconstruction.ipynb` to Colab, or place it directly in your Drive folder and open it from there.

### Step 3 — Configure the two settings in Cell 1

```python
REPO_DIR    = '/content/drive/My Drive/microscopy_data'   # path to your project folder on Drive
SCAN_FOLDER = 'my-scan_1'                                  # name of the subfolder containing the .ome.tif files
```

`FILE_PREFIX` is derived automatically from `SCAN_FOLDER` — no need to change it unless your files use a non-standard naming convention.

### Step 4 — Run all cells in order

| Cell | What it does |
|------|--------------|
| 1 | Mounts Drive, sets paths |
| 2 | Confirms the data folder exists |
| 3 | Installs `tifffile` |
| 4 | Imports all libraries |
| 5 | Reads metadata (grid size, stage positions, image dimensions) from the first file |
| 6 | Reads the full image stack and builds the spectral cube |
| 7 | Plots the integrated 2D intensity map |
| 8 | Interactive spectrum viewer (row/col sliders) |
| 9 | *(Optional)* Wavelength calibration |
| 10 | *(Optional)* Band-integrated map |
| 11 | Saves the spectral cube as `.npz` to Drive |

### Step 5 — (Optional) Wavelength calibration

If you have a dispersion calibration for your spectrometer, fill in Cell 9:

```python
WAVELENGTH_START_NM = 400.0   # wavelength at pixel 0, in nm
NM_PER_PIXEL        = 0.5     # dispersion in nm/pixel
```

Leave both as `None` to keep pixel indices on the X axis.

### Step 6 — (Optional) Band-integrated map

To highlight a specific spectral feature, set the pixel range in Cell 10:

```python
BAND_START_PX = 200   # start of integration window (pixel index)
BAND_END_PX   = 400   # end of integration window (pixel index)
```

---

## Output

- **Interactive 2D map** — integrated intensity across all wavelengths, with physical µm axes
- **Interactive spectrum viewer** — click through any grid position using sliders
- **`spectral_cube.npz`** saved in your scan folder, containing:

| Array | Shape | Description |
|-------|-------|-------------|
| `spectral_cube` | `(n_rows, n_cols, n_wavelengths)` | Full hyperspectral data cube |
| `intensity_map` | `(n_rows, n_cols)` | Mean intensity per pixel |
| `stage_x_um` | `(n_cols,)` | Stage X coordinates in µm |
| `stage_y_um` | `(n_rows,)` | Stage Y coordinates in µm |
| `wavelengths` | `(n_wavelengths,)` | Wavelength axis (nm or pixel index) |

To reload the cube in a later session:

```python
import numpy as np
data = np.load('spectral_cube.npz')
spectral_cube = data['spectral_cube']
stage_x_um    = data['stage_x_um']
stage_y_um    = data['stage_y_um']
wavelengths   = data['wavelengths']
```

---

## Troubleshooting

**`Exists: False` in Cell 2** — Check that `REPO_DIR` and `SCAN_FOLDER` together form the correct path. Drive must be mounted first (Cell 1).

**`Found 0 position files`** — The file prefix doesn't match. Check that `SCAN_FOLDER` matches the prefix of your `.ome.tif` filenames exactly, including capitalisation.

**Grid looks transposed** — Stage X maps to columns and stage Y maps to rows. If your scan is bidirectional and the image looks mirrored, the coordinate ordering in the metadata is still correct; the visual orientation just depends on which axis your stage scanned first.

**Slow loading** — Reading directly from Google Drive can be slow for large scans (100+ positions). The notebook reads the entire stack from a single file in one pass, so it is already optimised. If speed is still a concern, copy the scan folder to Colab's local SSD (`/content/`) before running Cell 6.

---

## Notes on the MicroManager OME-TIF format

MicroManager stores the complete acquisition metadata (grid size, number of positions, stage coordinates, pixel dimensions) in the header of every `.ome.tif` file. It also stores the **entire image stack** in the first file. This notebook exploits this: it reads the stack once with `tifffile.imread(first_file)` and iterates over frames in memory, which is far faster than opening each file individually.

---

## License

MIT — free to use, modify, and distribute.
