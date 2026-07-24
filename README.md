# MICROSCAN: FTIR Spectral Database Processing and Analysis

[![Data DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.15277130.svg)](https://doi.org/10.5281/zenodo.15277130)

This repository contains the code and supporting files used to process the MICROSCAN Fourier-transform infrared (FTIR) spectral collection, assemble the unified spectral database, apply a pretrained one-dimensional convolutional neural network (CNN1D), perform baseline correction, and reproduce the figures and supplementary visualizations associated with the project.

The metadata table distributed with the repository contains **2,010 spectra** assigned by spectral matching to three polymer classes: polyethylene (PE), polypropylene (PP), and polystyrene (PS).

## Contents

The workflow is organized into numbered directories that should normally be executed in order:

```text
.
├── 0_initial_sp_files/                         # Multipart 7z archive volumes and extracted raw .sp spectra
├── 1_converting_sp_to_csv/
│   ├── converting_sp_to_csv.ipynb
│   └── reference.sp
├── 2_compiling_unified_database/
│   ├── compiling_unified_database.ipynb
│   └── spectral_matching_classification.csv
├── 3_applying_CNN1D/
│   ├── CNN1D_application.ipynb
│   ├── cnn1d_v0.2.0.onnx.7z.001
│   ├── cnn1d_v0.2.0.onnx.7z.002
│   ├── ...
│   ├── cnn1d_results_full_db.pickle
│   ├── data/
│   └── src/
├── 4_baseline_correction/
│   ├── baseline_correction_Kedzierski.ipynb
│   └── R_packages/
└── 5_generating_figures/
    ├── Figure_1/
    ├── Figure_2/
    ├── Figure_3/
    ├── Figure_4/
    ├── Figure_5/
    ├── for_SI_static_spectra_all/
    └── for_Zenodo_interactive_spectra_all/
```

Generated files and Jupyter checkpoint directories are omitted from this overview.

## Raw spectra

The directory `0_initial_sp_files/` is included in the repository and contains the raw spectra as a multipart 7-Zip archive:

```text
0_initial_sp_files/0_initial_sp_files.7z.001
0_initial_sp_files/0_initial_sp_files.7z.002
0_initial_sp_files/0_initial_sp_files.7z.003
...
```

All archive volumes are parts of the same archive. Keep all parts in `0_initial_sp_files/`, do not rename them, and start extraction from the `.001` file only. The `.002`, `.003`, and subsequent files cannot be opened independently.

The raw spectra are also available from the associated MICROSCAN data record on Zenodo:

- <https://doi.org/10.5281/zenodo.15277130>

After extraction, the `.sp` files must be located directly inside `0_initial_sp_files/`, alongside the archive volumes:

```text
0_initial_sp_files/
├── 0_initial_sp_files.7z.001
├── 0_initial_sp_files.7z.002
├── ...
├── spectrum_0001.sp
├── spectrum_0002.sp
└── ...
```

The resulting CSV filenames must correspond to the values in the `File name` column of `2_compiling_unified_database/spectral_matching_classification.csv`.

## Software requirements

The notebooks were created with **Python 3.8.19**. A Python 3.8 environment is recommended for reproducing the original workflow. The code is designed primarily for Linux-like environments; Windows users may prefer WSL because several notebooks contain shell commands such as `mkdir`, `cp`, and `mv`.

### Core Python dependencies

- JupyterLab or Jupyter Notebook
- NumPy
- pandas
- Matplotlib
- tqdm
- specio
- SciPy
- scikit-learn
- Pillow
- ONNX Runtime
- Plotly
- Kaleido
- Cartopy
- pyproj
- rpy2
- openpyxl
- pyarrow

A possible Conda-based setup is:

```bash
conda create -n microscan python=3.8 jupyterlab r-base -c conda-forge
conda activate microscan

conda install -c conda-forge \
    numpy pandas matplotlib tqdm scipy scikit-learn pillow pyarrow \
    onnxruntime plotly kaleido cartopy pyproj rpy2 openpyxl

pip install specio
```

For systems without internet access during map generation, Natural Earth data can be installed in advance:

```bash
conda install -c conda-forge cartopy_offlinedata
```

The source files in `3_applying_CNN1D/src/` also include model-training and model-export utilities. These are not required for applying the supplied ONNX model. Running those optional utilities additionally requires packages such as PyTorch, PyTorch Lightning, torchmetrics, einops, ONNX, and skl2onnx.

### R requirements

Baseline correction uses R through `rpy2`. R must therefore be installed and visible from the same environment in which Jupyter is running. A C/C++/Fortran build toolchain may also be required because the supplied R packages are installed from source archives.

The notebook installs the required R packages from `4_baseline_correction/R_packages/` into `~/Rlibs` in the following order:

1. `SparseM`
2. `quadprog`
3. `lpSolve`
4. `limSolve`
5. `baseline`

## Running the workflow

> **Important:** run each notebook with its own directory as the current working directory. The notebooks use relative paths to exchange files between workflow stages.

### 1. Convert the original spectra to CSV

#### 1.1. Extract the multipart archive

On Ubuntu, install 7-Zip support if it is not already available:

```bash
sudo apt update
sudo apt install p7zip-full
```

From the project root, enter the directory containing the archive volumes:

```bash
cd 0_initial_sp_files
```

Verify that all archive parts are present:

```bash
ls -1 0_initial_sp_files.7z.*
```

Optionally test the complete multipart archive before extraction:

```bash
7z t 0_initial_sp_files.7z.001
```

Extract the spectra directly into the current `0_initial_sp_files/` directory:

```bash
7z e 0_initial_sp_files.7z.001 -o. -y
```

Use the `e` command rather than `x` here. The archive was created from the directory `0_initial_sp_files/`; therefore, `7z x` would preserve that stored directory name and could create an unwanted nested path:

```text
0_initial_sp_files/0_initial_sp_files/
```

The `e` command removes the stored directory prefix and places the `.sp` files directly alongside the archive volumes.

Only the `.001` file should be passed to 7-Zip. The program automatically reads `.002`, `.003`, and all subsequent parts. All volumes are required for successful extraction.

Confirm that the spectra were extracted into the expected location:

```bash
find . -maxdepth 1 -type f -name '*.sp' | wc -l
```

You can also inspect several filenames:

```bash
find . -maxdepth 1 -type f -name '*.sp' | sort | head
```

#### 1.2. Run the conversion notebook

Return to the project root, create the output directory, and start the notebook from `1_converting_sp_to_csv/`:

```bash
cd ../1_converting_sp_to_csv
mkdir -p csv_files
jupyter lab converting_sp_to_csv.ipynb
```

The notebook:

- reads the extracted spectra from `../0_initial_sp_files/`;
- reads `reference.sp` to define the expected wavenumber grid;
- converts the original signal to absorbance;
- verifies that every spectrum uses the reference grid;
- writes one CSV file per spectrum to `csv_files/`.

Expected output:

```text
1_converting_sp_to_csv/csv_files/*.csv
```

### 2. Compile the unified MICROSCAN database

```bash
cd ../2_compiling_unified_database
jupyter lab compiling_unified_database.ipynb
```

The notebook combines the converted spectra with the manual spectral-matching metadata in `spectral_matching_classification.csv`.

Expected outputs:

```text
2_compiling_unified_database/MICROSCAN_database.csv
2_compiling_unified_database/MICROSCAN_database.xlsx
```

The database contains descriptive metadata followed by the absorbance values on the common wavenumber grid.

### 3. Apply the pretrained CNN1D model

#### 3.1. Restore the ONNX model from the multipart archive

The pretrained model is stored in `3_applying_CNN1D/` as a multipart 7-Zip archive:

```text
3_applying_CNN1D/cnn1d_v0.2.0.onnx.7z.001
3_applying_CNN1D/cnn1d_v0.2.0.onnx.7z.002
...
```

All archive volumes are parts of the same file. Keep them together in `3_applying_CNN1D/`, do not rename them, and start extraction from the `.001` volume only.

From the project root, run:

```bash
cd 3_applying_CNN1D
7z x cnn1d_v0.2.0.onnx.7z.001 -o. -y
```

7-Zip automatically reads `.002` and any subsequent volumes. All parts are required. After successful extraction, the following file must exist directly in `3_applying_CNN1D/`:

```text
3_applying_CNN1D/cnn1d_v0.2.0.onnx
```

Verify the reconstructed model:

```bash
ls -lh cnn1d_v0.2.0.onnx
```

Optionally test the multipart archive before extraction:

```bash
7z t cnn1d_v0.2.0.onnx.7z.001
```

A successful test ends with `Everything is Ok`.

#### 3.2. Run the CNN1D notebook

If you are already in `3_applying_CNN1D/` after restoring the model, start the notebook with:

```bash
jupyter lab CNN1D_application.ipynb
```

Alternatively, from `2_compiling_unified_database/`, run:

```bash
cd ../3_applying_CNN1D
jupyter lab CNN1D_application.ipynb
```

The notebook applies the reconstructed `cnn1d_v0.2.0.onnx` model to every converted spectrum by calling `python -m src.infer`. Inference uses the CPU by default.

Before execution, verify that the `red_db_file` variable points to the database created in Step 2. With the directory structure shown above, the path is:

```python
red_db_file = "../2_compiling_unified_database/MICROSCAN_database.csv"
```

Expected outputs:

```text
3_applying_CNN1D/cnn1d_results_full_db.pickle
3_applying_CNN1D/manual_and_CNN1D_classification.csv
```

A precomputed `cnn1d_results_full_db.pickle` is included in the repository and may be reused when model inference does not need to be repeated.

A single spectrum can also be processed directly from the terminal:

```bash
python -m src.infer \
    -i ../1_converting_sp_to_csv/csv_files/<spectrum>.csv \
    -m cnn1d_v0.2.0.onnx \
    --model-name cnn \
    -k 1
```

### 4. Correct spectral baselines

```bash
cd ../4_baseline_correction
jupyter lab baseline_correction_Kedzierski.ipynb
```

On the first run, execute the R-package installation section. The notebook then applies `baseline.fillPeaks` to each spectrum and constrains negative corrected values to zero.

Expected output:

```text
4_baseline_correction/MICROSCAN_database_baseline_corrected.csv
```

### 5. Generate figures and supplementary visualizations

Steps 2–4 must be completed before running the spectral classification figures. Launch each notebook from the directory in which it is stored.

| Directory | Notebook | Main output |
|---|---|---|
| `5_generating_figures/Figure_1/` | `draw_map.ipynb` | Arctic sampling map in PNG, PDF, and SVG formats under `map_outputs/` |
| `5_generating_figures/Figure_2/` | `draw_DSC_curves.ipynb` | `DSC_panels_highres.png` |
| `5_generating_figures/Figure_3/` | `draw_sankey_plots.ipynb` | Separate PE, PP, and PS Sankey diagrams |
| `5_generating_figures/Figure_4/` | `draw_figure_4.ipynb` | Averaged spectra for selected PE+PA, PE+PVC, and PE+CPE predictions |
| `5_generating_figures/Figure_5/` | `draw_figure_5.ipynb` | Averaged spectra for selected PE+EVA and PE+PU predictions |
| `5_generating_figures/for_SI_static_spectra_all/` | `draw_static_spectra_all.ipynb` | Static PNG spectra for all true/predicted class combinations |
| `5_generating_figures/for_Zenodo_interactive_spectra_all/` | `draw_interactive_spectra_all.ipynb` | Interactive Plotly HTML spectra for all true/predicted class combinations |

To avoid shell warnings in the supplementary-output notebooks, the output directories may be created before execution:

```bash
mkdir -p 5_generating_figures/for_SI_static_spectra_all/png_files
mkdir -p 5_generating_figures/for_Zenodo_interactive_spectra_all/html_files
```

The map notebook uses Natural Earth vector data through Cartopy. If the required layers are not already cached, Cartopy may download them during the first run.

## Input and output summary

| Stage | Input | Output |
|---|---|---|
| Spectrum conversion | Multipart archive volumes in `0_initial_sp_files/`, extracted `.sp` files, and `reference.sp` | Individual spectral CSV files |
| Database compilation | Spectral CSV files and manual metadata | Unified CSV and XLSX databases |
| CNN1D inference | Unified database, spectral CSV files, ONNX model | Per-spectrum model predictions |
| Baseline correction | Unified database | Baseline-corrected database |
| Figure generation | Databases, CNN predictions, coordinates, DSC files | Publication and supplementary figures |

## Reproducibility notes

- Relative paths are used throughout the notebooks; changing the directory names requires updating those paths.
- `reference.sp` defines the expected spectral grid. Spectra on a different grid are reported and not exported by the conversion notebook.
- Before CNN1D inference, reconstruct `cnn1d_v0.2.0.onnx` from the multipart archive in `3_applying_CNN1D/`. The extracted model and label mapping files must remain in their expected locations relative to `CNN1D_application.ipynb`.
- The figure notebooks assume that the database, baseline-corrected database, and CNN classification table have already been generated.
- Plotly static image export requires a working Kaleido installation.
- The repository does not currently include a dependency lock file. For strict reproducibility, record the exact package versions from the environment used for the final analysis.

## Data availability

The source spectra and the associated MICROSCAN data products are available from Zenodo:

- **MICROSCAN database:** <https://doi.org/10.5281/zenodo.15277130>

The raw `.sp` spectra are provided in this GitHub repository as multipart 7-Zip archive volumes under `0_initial_sp_files/`. Extract them into the same directory before running the full workflow. The source spectra and associated data products are also preserved in the Zenodo record linked above.

## Citation

When using this repository, please cite both the associated publication and the archived MICROSCAN dataset:

- MICROSCAN database, Zenodo: <https://doi.org/10.5281/zenodo.15277130>

<!-- Citation for the associated Scientific Data article will be here after publication. -->

## License and third-party components

A project-level license is not included in the supplied archive. Add an appropriate `LICENSE` file before public release and update this section accordingly.

The Python files under `3_applying_CNN1D/src/` contain their own author and Apache License 2.0 notices. Those notices should be retained when redistributing or modifying that code.
