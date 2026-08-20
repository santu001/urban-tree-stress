# Urban Tree Stress Detection — Frankfurt am Main

Code and data for the M.Sc. thesis *"Remote Sensing-Based Differentiation of Tree Stress Factors in Urban Environments Using Multispectral and SAR Data: A Case Study in Frankfurt am Main"* (Santosh Ganiger, Frankfurt UAS, in co-operation with Karuna Technology).

**What it does in one sentence:** it takes leaf-level health measurements from a handheld device (Arborcheck) and satellite reflectance from PlanetScope for the same trees on the same days, joins them into one table, and trains classifiers to predict a tree's **vitality** and **stress level** from the satellite data alone.

---

## 1. The idea behind the pipeline

Most remote sensing tools can tell you *that* a tree is stressed. This project tries to tell you *why* (drought vs. nutrient), so the city can decide whether to water or fertilise.

To train a model for that, you need two things measured on the same tree on the same day:

| Source | What it gives you | Role |
|---|---|---|
| **Arborcheck** (handheld, in the field) | Fv/Fm efficiency, chlorophyll content, stress indices Si1–Si4 | **Labels** (ground truth) |
| **PlanetScope** (satellite, 8-band, ~3 m) | Reflectance in 8 bands B1–B8 | **Features** (model inputs) |

The labels are continuous numbers, so they are converted into categories with fixed rules, and those categories become the two prediction targets:

- `VitalityCategory` → `Good` / `Slight reduction` / `Critical reduction`
- `StressCategory` → `No Stress` / `Mild` / `Moderate` / `Severe`

---

## 2. End-to-end data flow

```
FIELD                                    SATELLITE
─────                                    ─────────
Arborcheck device                        [GEE script A] tree detection
   │  64 × .res files                     → polygons → shapefile export
   │                                             │
   │                                             ▼
   │                                     Planet API ──► GEE Image Collection
   │                                     (Karuna's script — contact Karuna
   │                                      for IC access for any new AOI)
   │                                             │
   ▼  [notebook 1, cell 1]                       ▼  cloud/shadow mask, radiometric
all_res_combined.csv                                normalisation, clip to AOI
   │                                             │
   │                                             ▼  [GEE script B] zonal mean
   ▼  [notebook 1, cell 3]                          per tree polygon, B1–B8
Arborcheck_with_categories.csv                 FraOst_Planet_Tree_Data.csv
(adds VitalityCategory + StressCategory)       (BaumID, Date, B1..B8)
   │                                             │
   └──────────────► [notebook 1, cell 7] ◄───────┘
                    merge on BaumID + Date
                              │
                              ▼
                Data/Data_To_Train_3_vitality.xlsx   ← 86 rows, the model-ready table
                              │
      ┌───────────────────────┼───────────────────────┐
      ▼                       ▼                       ▼
 Logistic Regression    Linear SVM (calibrated)   Random Forest
```

Both halves are keyed on **`BaumID` + `Date`**. That join is the heart of the whole project — everything else is preparation for it.

---

## 3. Repository contents

```
urban-tree-stress/
├── Data_Reading,_Cleaning_&_Tranformation.ipynb   # Step 1: build the dataset
├── Logistic_regression_two_targets_with_graphs.ipynb
├── Linear_svm_calibrated_two_targets_with_full_indices_and_graphs.ipynb
├── RF_Model_two_targets_with_full_indices_and_graphs.ipynb
└── Data/
    ├── Data_To_Train_3_vitality.xlsx     # ★ final training table (86 rows × 21 cols)
    ├── Arborcheck_Tree_Data/             # 64 raw .res files from the device
    ├── Fra_Ost_Selected_Trees.csv        # tree inventory from Grünflächenamt
    └── Frankfurt_Trees_Shp/
        ├── Fra_1/  Fra_Trees_1.shp       # 4 trees: BaumID 801, 808, 809, 749
        ├── Fra_2/  Fra_Trees_2.shp       # 4 trees: BaumID 444, 578, 571, 569
        └── Fra_3/  Fra_Trees_3.shp       # historical trees: 152, 153, 154, 161
```

### `Data_To_Train_3_vitality.xlsx` — the one file that matters most

86 rows, one per **tree × date** observation. 12 trees, August 2025 campaign plus historical records.

| Column | Meaning |
|---|---|
| `Date`, `BaumID` | join keys |
| `Genus`, `Species` | e.g. Quercus robur |
| `Efficiency`, `Chlorophyll` | raw Arborcheck readings → drive `VitalityCategory` |
| `Si1`–`Si4` | Arborcheck stress indices → drive `StressCategory` |
| `VitalityCategory`, `StressCategory` | **the two prediction targets** |
| `FID` | polygon ID from the shapefile |
| `B1`–`B8` | mean PlanetScope reflectance over the tree's canopy polygon |

Band mapping (PlanetScope SuperDove):

| | B1 | B2 | B3 | B4 | B5 | B6 | B7 | B8 |
|---|---|---|---|---|---|---|---|---|
| | Coastal Blue | Blue | Green I | Green | Yellow | Red | **Red Edge** | **NIR** |
| nm | 443 | 490 | 531 | 565 | 610 | 665 | 705 | 865 |

B7 and B8 do most of the work — red-edge and NIR are where chlorophyll loss shows up first.

Class counts (note the imbalance — this is the project's main weakness):

```
VitalityCategory: Good 72 | Critical reduction 8 | Slight reduction 6
StressCategory:   Mild 50 | Severe 23 | No Stress 7 | Moderate 6
```

---

## 3a. Google Earth Engine scripts (run these first, before the notebooks)

The satellite half of the pipeline lives in GEE, not in this repo. Two scripts do the work. Open them in the **GEE Code Editor** — you need a Google account with Earth Engine access, and the imagery must already be ingested into your GEE assets.

### Script A — Tree detection and shapefile export

**https://code.earthengine.google.com/8eebe467c21cf23c7fe36564617c3e2c**

Turns tree locations into the polygon geometries everything else keys off.

What it does:
1. Locates each study tree on the satellite basemap using the GPS coordinates recorded in the field.
2. Draws a polygon around each tree canopy — this is the **Area of Interest** whose pixels get averaged later.
3. Tags each polygon with a unique `FID` and its `BaumID`, so it matches the field records one-to-one.
4. Collects the polygons into a `FeatureCollection` and computes extra attributes (area, centroid lat/lon).
5. **Exports the result as a shapefile** to Google Drive.

The exported shapefiles are already committed here as `Data/Frankfurt_Trees_Shp/Fra_1|Fra_2|Fra_3`. Rerun this script only when adding new trees.

> The polygon boundary matters more than it looks. Draw it too big and you average in grass, path and shadow; too small and you get too few pixels. Misaligned polygons are the most likely source of silently bad reflectance values.

### Script B — 8-band spectral extraction per tree

**https://code.earthengine.google.com/c7a7bae7aa5f67253f3a3ae827632877**

Turns imagery plus polygons into the `B1`–`B8` numbers used for training. This is the step that produces `FraOst_Planet_Tree_Data.csv`.

What it does:
1. Loads the post-processed PlanetScope Image Collection (`GDE_8B_FRA_PSScene1/2/3`) and the tree polygon shapefile from Script A.

> **Working on a new area?** The Image Collection must exist before this script can run. **Contact Karuna Technology to request Image Collection access for your AOI** — they hold the Planet API key and run the ingestion. Give them the bounding box and the date range you need; they create and populate the collection, then you point Script B at it.
2. Filters the collection:
   - **spatially** — only scenes intersecting the tree polygons
   - **temporally** — restricted to the field campaign window
   - **by quality** — `local_clear_percent ≥ 0.99`, `local_mean_clear_confidence_percent ≥ 80`, `publishing_stage = finalized`, `quality_category = standard`, `image_order_in_date = 1`
3. For each tree polygon and each surviving scene, computes the **mean pixel value of all eight bands** over the polygon (zonal statistics, mean reducer, fixed at 3 m scale).
4. Builds one record per tree per date: `FID`, `BaumID`, `Date`, `TreeDetails`, `ShapeID`, `B1`–`B8`.
5. Processes in batches (1,000 features at a time) to stay inside GEE's limits, then **exports CSV to Google Drive**.

Download the exported CSV, name it `FraOst_Planet_Tree_Data.csv`, and feed it to cell 7 of notebook 1.

### What is still missing

Script B assumes the imagery is **already in your GEE Image Collection**. Getting it there — authenticating to the Planet API, searching and ordering scenes, downloading, harmonising, ingesting — is **Karuna Technology's proprietary script and is not available**. Thesis §3.4.6 describes its logic but not its code.

**To work on any new AOI, contact Karuna Technology and request Image Collection access.** They hold the Planet API key and run the ingestion on their side. What to send them:

- the **bounding box** of your area of interest (and its four corner points)
- the **date range** you need imagery for
- the **product**: PlanetScope SuperDove 8-band PSScene surface reflectance, plus the UDM2 quality mask
- a **name for the collection**, following the existing convention (`GDE_8B_FRA_PSScene1/2/3`)

Once they've populated the collection and shared it with your GEE account, Script B runs against it unchanged.

So the runnable boundary is:

| Step | Available? |
|---|---|
| Tree detection → shapefile | ✅ Script A |
| Planet API → GEE Image Collection | ❌ Karuna proprietary — **contact Karuna for IC access per AOI** |
| Image Collection + polygons → B1–B8 CSV | ✅ Script B |
| CSV → merged table → models | ✅ this repo |

Only the middle step is blocked. If someone else ingests imagery into a GEE collection with the same band layout, the rest of the chain runs.

---

## 4. Setup

Everything was written for **Google Colab**, but runs fine locally.

### Local

```bash
git clone https://github.com/santu001/urban-tree-stress.git
cd urban-tree-stress

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install pandas numpy openpyxl scikit-learn matplotlib seaborn joblib jupyter
jupyter notebook
```

Verified working on Python 3.11–3.12, scikit-learn 1.8, pandas 2.x.

> **One compatibility fix is required** — see [Known issues](#7-known-issues--gotchas) before running the Logistic Regression notebook.

### Colab

Upload the `Data/` folder (or mount Drive), then fix the paths as described below.

---

## 5. Running the code

### Path variables you must change

Every notebook hardcodes Colab paths (`/content/...`). Update these first:

| Notebook | Line to edit | Change to |
|---|---|---|
| Data Reading | `convert_folder("/content", ...)` | `convert_folder("Data/Arborcheck_Tree_Data", "all_res_combined.csv")` |
| Data Reading | `input_csv` / `output_csv` | local paths |
| Data Reading | `csv_path_1`, `csv_path_2` | local paths |
| Logistic Regression | `DATA_PATH` | `"Data/Data_To_Train_3_vitality.xlsx"` |
| Linear SVM | `DATA_PATH` | `"Data/Data_To_Train_3_vitality.xlsx"` |
| Random Forest | `DATA_PATH` | `"Data/Data_To_Train_3_vitality.xlsx"` |

### Shortcut: skip straight to modelling

`Data/Data_To_Train_3_vitality.xlsx` is already built and committed. **To just reproduce the thesis results, change `DATA_PATH` in the three model notebooks and run them.** Notebook 1 is only needed if you're adding new field data.

---

### Notebook 1 — `Data_Reading,_Cleaning_&_Tranformation.ipynb`

Builds the training table. Four independent cells, run in order.

**Cell 1 — parse the `.res` files.**
Each `.res` file is one measurement session for one tree. Format:

```
5                                    ← number of value lines
0.0,1                                ← value, quality flag (1 = valid)
0.0,1
0.0,1
-1.4688091674366188,1
-0.9847641843512683,1
-0.7280149430922292,1
749,13,8,25,12,11,Quercus,robur,*MEASURED*,...,50.1212984300,8.7222980400
11327                                ← session ID
```

The metadata line is `BaumID, day, month, year, hour, minute, genus, species, status, ..., lat, lon`. The six values map to `Efficiency, Chlorophyll, Si1, Si2, Si3, Si4`. The date is taken from the **filename** (`ARB-749-13-08-25_12-11.res` → 13/08/2025), not the metadata line.

Output: `all_res_combined.csv`, 64 rows.

**Cell 3 — turn numbers into categories.** This is where the labels are created, using fixed thresholds:

```python
# Vitality — from Efficiency, refined by Chlorophyll
Efficiency < -4.0                 → "Critical reduction"
-4.0 ≤ Efficiency < -3.0          → "Significant reduction"
-3.0 ≤ Efficiency < -2.0          → "Slight reduction"
Efficiency ≥ -2.0                 → "Good"
# ...but "Good" is downgraded to "Slight reduction" if Chlorophyll < -2.5

# Stress — each Si gets a severity band 1–3, then:
any band == 3                     → "Severe"
worst band == 2 and |mean| ≥ 1.5  → "Moderate"
worst band == 2                   → "Mild"
otherwise                         → "Mild" or "No Stress"
```

Deliberately conservative: one strongly deviating index is enough to call a tree `Severe`.

Output: `Arborcheck_with_categories.csv`.

**Cell 7 — merge with satellite data.** Inner join on `BaumID` + `Date`. Requires `FraOst_Planet_Tree_Data.csv`, which is **not in this repo** (see [Known issues](#7-known-issues--gotchas)).

---

### Notebooks 2–4 — the models

All three share the same skeleton: load Excel → compute spectral indices → stratified train/test split → fit → evaluate → save plots and `.joblib` artifacts. Each trains **two independent models**, one per target.

| | Logistic Regression | Linear SVM | Random Forest |
|---|---|---|---|
| Estimator | multinomial, saga | `LinearSVC` + `CalibratedClassifierCV` | 500 trees, `min_samples_leaf=2` |
| Scaling | `StandardScaler` | `StandardScaler` | none (scale-invariant) |
| Features | 8 bands + 16 indices + 8 log1p = **32** | same = **32** | 8 bands + 16 indices = **24** |
| Split | 75/25 stratified | 75/25 stratified | **80/20** stratified |
| Class balancing | `class_weight="balanced"` | `class_weight="balanced"` | `balanced_subsample` |
| Output folder | `logreg_outputs/` | `lsvm_outputs/` | `model_artifacts_extended/` |
| `random_state` | 42 | 42 | 42 |

**Spectral indices computed from the raw bands** (this is the feature engineering — no new data, just ratios that amplify physiological signal):

- *Greenness / structure:* NDVI, GNDVI, EVI, SAVI, MSAVI2, RDVI
- *Chlorophyll / nutrient:* NDRE, CIre, CIgreen, MTCI
- *Photochemical:* PRI
- *Ratios & other:* VARI, NDWI, RE/Red, Red/NIR, Blue/Red

Each notebook writes three plots per target: confusion matrix, top-20 feature importance, per-class precision/recall/F1.

---

## 6. Expected results

Running the notebooks as committed reproduces the thesis numbers **exactly** (all seeds fixed at 42):

| Model | Target | Accuracy | Balanced Acc | Macro-F1 |
|---|---|---|---|---|
| Logistic Regression | Vitality | 0.7273 | 0.4444 | 0.3775 |
| Logistic Regression | Stress | 0.5455 | 0.4263 | 0.3929 |
| Linear SVM | Vitality | 0.8182 | 0.3333 | 0.3000 |
| Linear SVM | Stress | 0.8182 | 0.4583 | 0.4325 |
| Random Forest | Vitality | 0.7222 | 0.2889 | 0.2889 |
| **Random Forest** | **Stress** | **0.8333** | 0.4500 | 0.4422 |

If you get different numbers, something in the environment or data path changed.

**How to read these honestly:** accuracy looks decent, but balanced accuracy and macro-F1 are low. The models are good at `Good` vitality and at `Mild`/`Severe` stress; they essentially cannot predict the minority classes (`Moderate`, `No Stress`, `Slight`/`Critical reduction`). With a test set of only 18–22 samples, some classes have 1–2 examples. Quote accuracy only alongside macro-F1.

Consistently important features across all three models: **NDRE, MTCI, CIre, EVI, VARI, RE/Red, Blue/Red** — all red-edge and NIR driven, which is physiologically what you'd expect. That's a good sign the models learned real signal, not noise.

---

## 7. Known issues & gotchas

**Read this section before changing anything.**

1. **`multi_class` breaks on scikit-learn ≥ 1.7.** The Logistic Regression notebook fails with `TypeError: LogisticRegression.__init__() got an unexpected keyword argument 'multi_class'`. Fix — delete this one line from `make_model()`:
   ```python
   multi_class="multinomial",   # ← remove; multinomial is the default for saga
   ```
   Results are unchanged after the fix. Alternatively pin `scikit-learn<1.7`.

2. **The Planet API download step is proprietary to Karuna Technology and is not available.** Tree detection (Script A) and 8-band extraction (Script B) *are* available in GEE — see [§3a](#3a-google-earth-engine-scripts-run-these-first-before-the-notebooks). What's missing is the middle step that authenticates to Planet, orders scenes and ingests them into the GEE Image Collection. **For any new AOI, contact Karuna Technology to request Image Collection access** — send them the bounding box, date range and collection name, and they populate it on their side. You can rerun extraction freely over imagery that's already ingested.

3. **`FraOst_Planet_Tree_Data.csv` is missing from the repo.** Cell 7 of notebook 1 will fail. The already-merged output (`Data_To_Train_3_vitality.xlsx`) is committed, so modelling still works.

4. **The vitality class merge is not in the code.** The notebook produces **four** vitality classes, but the training Excel has **three** — `Significant reduction` was merged into `Critical reduction` manually to reduce class imbalance (thesis §3.5.3, §4.2.4). If you rerun notebook 1 you must redo this merge by hand, or the model notebooks will train on four classes and results won't match.

5. **Band values are scaled integers, not 0–1 reflectance.** `B1`–`B8` in the Excel range roughly 300–3000 (reflectance × 10000). Ratio indices (NDVI, NDRE, CIre...) are unaffected. But **EVI, SAVI, MSAVI2 and RDVI contain additive constants** (`+1.0`, `+0.5`) that assume 0–1 reflectance — at this scale those constants are numerically negligible, so those four indices don't mean quite what their formulas say. They still work as learned features, but don't interpret their absolute values. Dividing bands by 10000 before index computation would be the correct fix.

6. **Texture features never activate.** `TEXTURE_PATCH_COLS` looks for `NDVI_patch`, `B7_patch`, `B8_patch` columns that don't exist in the Excel, so the RF always runs with 24 features and `texture=0`. The code path is dead but harmless.

7. **Random Forest uses `TEST_SIZE = 0.2` while LR and SVM use `0.25`.** Cross-model comparison isn't strictly apples-to-apples. Worth standardising.

8. **86 rows, 12 trees.** Observations from the same tree on consecutive days are highly correlated, and a random split puts some of them in train and some in test — so the reported scores are likely optimistic. A grouped split by `BaumID` would be more honest.

9. Filenames contain a typo (`Tranformation`) and a comma, which needs quoting in shell commands: `jupyter nbconvert "Data_Reading,_Cleaning_&_Tranformation.ipynb"`.

10. No `requirements.txt`, no random-seed documentation outside the notebooks, no unit tests.

---

## 8. External resources

| Resource | Link / location |
|---|---|
| **GEE Script A — tree detection & shapefile export** | https://code.earthengine.google.com/8eebe467c21cf23c7fe36564617c3e2c |
| **GEE Script B — 8-band extraction per tree** | https://code.earthengine.google.com/c7a7bae7aa5f67253f3a3ae827632877 |
| GEE script — polygon delineation (thesis §3.3) | https://code.earthengine.google.com/5c305ce62bdec2f9d301a7bdeb513b45 |
| GEE Image Collections | `GDE_8B_FRA_PSScene1` / `PSScene2` / `PSScene3` |
| **Image Collection access for a new AOI** | **contact Karuna Technology** (they hold the Planet API key and run ingestion) |
| Planet API key | held by Karuna Technology |
| Field data & Arborcheck device | Grünflächenamt, City of Frankfurt am Main |

Study area: Ostpark, Frankfurt am Main. Species: *Ilex aquifolium*, *Fagus sylvatica*, *Carpinus betulus*, *Quercus robur*, *Tilia cordata*.

---

## 9. Suggested next steps

Ordered roughly by value per unit of effort:

1. **Collect more data, especially minority classes.** The thesis is explicit that this is the binding constraint. Everything else is secondary.
2. Switch to **grouped cross-validation by `BaumID`** instead of a single random split — will lower the reported numbers, but they'll be trustworthy.
3. Rescale bands to 0–1 before computing EVI/SAVI/MSAVI2/RDVI.
4. Reimplement the Planet API ingestion step (the only remaining proprietary link) so the pipeline runs end-to-end independently. Until then, request Image Collection access from Karuna Technology for each new AOI.
5. Add Sentinel-1 SAR features — named in the thesis title and motivation, but not actually used in the final models.
6. Try temporal modelling (per-tree time series) instead of treating each tree-date as independent.

---

## 10. Credits

Santosh Ganiger — M.Sc. High Integrity Systems, Frankfurt University of Applied Sciences (Nov 2025).
Supervisors: Prof. Dr. Christina Andersson; Erik Kaiser (Karuna Technology).
Field data and equipment: Grünflächenamt, City of Frankfurt am Main.
Satellite imagery: Planet Labs PBC (PlanetScope SuperDove), via Karuna Technology.
