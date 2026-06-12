# Biotic_Poster



# Input data format

This document describes the structure expected by the notebook for every input file. Place all input files in the working directory next to the notebook.

## File inventory

| File | Required for | Source |
|---|---|---|
| `2025_RB_Hek293_ERG1_5xLB_Report.xlsx` | training (HEK293) | Spectronaut DIA glycoproteomics |
| `June_2024_HeLa_GFP_ERG1_2LB_SSL_ChRed_Report_Pivot_notNorm_Summ.csv` | training (HeLa) | Spectronaut DIA glycoproteomics |
| `2025_RB_Huh7_ERG1_5xLB_Report.xlsx` | training (Huh7) | Spectronaut DIA glycoproteomics |
| `2025_RB_MHC_ERG1_7xLB_Report.xlsx` | training (MHC) | Spectronaut DIA glycoproteomics |
| `human_proteoform_glycosylation_sites_glyconnect.csv` | training (canonical negatives) | GlyGen `GLY_000150` |
| `human_proteoform_glycosylation_sites_unicarbkb.csv` | training (canonical negatives) | GlyGen `GLY_000149` |
| `KPC47_WTG1_WT_ERG1_4LB_SSL_ChRed_Report_Pivot_notNorm_Summ.csv` | validation only | Spectronaut DIA glycoproteomics, mouse |
| `2025_RB_KIC_HMP_10xLB_Report.xlsx` | validation only | Spectronaut DIA glycoproteomics, mouse |

The notebook will refuse to start if any of the six required files is missing. The two validation files are optional.

---

## Spectronaut reports

Both `.xlsx` and `.csv` files are accepted. The expected schema is identical for all six Spectronaut reports (four human training reports plus the two mouse validation reports). All training files come from Spectronaut DIA glycoproteomics on Jacalin-enriched lysates, exported in the long peptide-level format.

### Required columns

Every Spectronaut report must contain at least these columns:

| Column | Type | Example | Notes |
|---|---|---|---|
| `PG.ProteinAccessions` | string | `P00533` or `A6NMY6;P07355` | UniProt accessions separated by `;` or `,`. The first token is used. Rows with multiple distinct accessions are kept; rows where `PEP.PeptidePosition` is also a list (one position per accession) are dropped because the position cannot be assigned unambiguously. |
| `PG.Genes` | string | `EGFR` | Optional metadata, kept in the glycoform table. |
| `PEP.StrippedSequence` | string | `LSLEGDHSTPPSAYGSVK` | The peptide sequence without modifications, uppercase, single-letter amino acids. |
| `PEP.PeptidePosition` | string or int | `11` or `11;11` | The position of the first residue of the peptide in the protein. If the peptide maps to multiple positions (separated by `;` or `,`), the row is dropped. |
| `EG.ModifiedSequence` | string | see below | The glycoform sequence with HexNAc tags. |
| Quantitative columns | float | see below | One column per sample, with the substring `MS2Quantity` in the column name. |

Additional Spectronaut columns are ignored. The order of columns does not matter.

### Quantitative column naming

The notebook identifies ERG1 (positive) and control (negative) samples by string matching the column names. The matching rules per cell line are hardcoded in `CELL_LINE_CONFIG` near the top of the cell loader:

```python
CELL_LINE_CONFIG = {
    "HEK293": {"erg1_pattern": "Hek293_ERG1", "ctrl_pattern": "Hek293_WT",   "ctrl_exclude": None,   "drop_patterns": []},
    "HeLa":   {"erg1_pattern": "HeLaERG1",    "ctrl_pattern": "HeLaGFP",     "ctrl_exclude": None,   "drop_patterns": []},
    "Huh7":   {"erg1_pattern": "Huh7_ERG1",   "ctrl_pattern": "Huh7_WT",     "ctrl_exclude": None,   "drop_patterns": ["lectin_TM"]},
    "MHC":    {"erg1_pattern": "MHC_ERG1",    "ctrl_pattern": "MHC_",        "ctrl_exclude": "ERG1", "drop_patterns": []},
}
```

For each cell line:
- Every column containing both the cell line `erg1_pattern` AND `MS2Quantity` becomes an ERG1 replicate.
- Every column containing both `ctrl_pattern` AND `MS2Quantity` becomes a control replicate. If `ctrl_exclude` is set, columns containing that substring are removed from the control list (used by MHC to separate WT from ERG1 since both start with `MHC_`).
- Columns containing any of `drop_patterns` are removed up front and never considered.

So an HEK293 report should have columns whose names look like:

```
[1] 240619_Lumos_nLC1200_Bard_RB_Hek293_ERG1_Hu_200ug_DIA_sLWAC_Jacalin_EL_5(20)_ul_1.raw.PEP.MS2Quantity
[2] 240619_Lumos_nLC1200_Bard_RB_Hek293_ERG1_Hu_200ug_DIA_sLWAC_Jacalin_EL_5(20)_ul_2.raw.PEP.MS2Quantity
...
[N] 240619_Lumos_nLC1200_Bard_RB_Hek293_WT_Hu_200ug_DIA_sLWAC_Jacalin_EL_5(20)_ul_1.raw.PEP.MS2Quantity
[N+1] 240619_Lumos_nLC1200_Bard_RB_Hek293_WT_Hu_200ug_DIA_sLWAC_Jacalin_EL_5(20)_ul_2.raw.PEP.MS2Quantity
```

Exact column count is flexible. Each cell line should have at least 2 ERG1 replicates and 2 CTRL replicates for the classification step to work. A peptide is classified as `present_ERG1` if it has a non-NaN intensity in `>= ceil(0.5 * n_replicates)` ERG1 columns, and similarly for CTRL. Intensity values may be NaN, empty, or zero for missing peptides; rows where all intensity columns are empty are dropped.

### `EG.ModifiedSequence` format

Two Spectronaut export styles are accepted. In both, HexNAc modifications appear as bracketed tags inside the sequence:

Format A, N-terminal localized (common in Jacalin-enriched DIA):
```
_[HexNAc(1) (Any N-term)]LSLEGDHSTPPSAYGSVK_
_[HexNAc(2) (Any N-term)]LSLEGDHSTPPSAYGSVK_
_[HexNAc(3) (Any N-term)]LSLEGDHSTPPSAYGSVK_
```

Format B, inline (common in directDIA):
```
_LS[HexNAc(1)]LEGDHSTPPSAYGSVK_
_LSL[HexNAc(1)]EGDH[HexNAc(1)]STPPSAYGSVK_
```

The notebook parses every `[HexNAc(N)...]` tag in the row and sums the `N` values to obtain the number of glycosylated sites per peptide (`n_glycans`). For example, `[Hex(1)HexNAc(2) ...]` contributes 2 sites, `[HexNAc(3) ...]` contributes 3 sites. A row with no `HexNAc` tag is dropped.

Rows where `n_glycans > n_ST` (more glycans than S or T residues in the peptide) are dropped as Spectronaut N-term mislocalization artefacts.

### Position handling

`PEP.PeptidePosition` is the 1-based index of the first residue of the peptide in the protein. The notebook only keeps peptides with a single integer position. Rows like:

```
PG.ProteinAccessions  PEP.PeptidePosition
P00533                312
A6NMY6;P07355         11;11
A6NMY6;P07355         11;22
```

are handled as follows:
- Row 1: kept, mapped to P00533 at position 312.
- Row 2: kept, both accessions map to the same start position. The first accession is used.
- Row 3: dropped (multi-mapping row, ambiguous coordinates).

---

## Validation reports

### KPC47 (mouse)

`KPC47_WTG1_WT_ERG1_4LB_SSL_ChRed_Report_Pivot_notNorm_Summ.csv`

Same column schema as the human reports. The classification regex is more permissive:
- ERG1: column name matches `ERG1` (case-insensitive)
- CTRL: column name matches `WT`, `CTRL` or `GFP` and is not ERG1
- Excluded entirely: columns containing `WTG1` (an intermediate condition that is not used as either ERG1 or WT)

UniProt accessions in this file are mouse (e.g. `Q9D880`, `P09581`).

### KIC vs HMP (mouse)

`2025_RB_KIC_HMP_10xLB_Report.xlsx`

Same schema again, but the classification regex differs because there is no ERG1 condition:
- KIC (tumor, treated as the GALA-positive class): column name matches `_KIC\d` (e.g. `_KIC1`, `_KIC2`, ...)
- HMP (healthy pancreas, treated as the Golgi-reference class): column name matches `_HMP\d`

The classification thresholds (50% replicate fraction, log2FC >= 3 for shared-enriched) are the same as for training.

---

## Adapting to a new cell line

To add a fifth training cell line, edit `CELL_LINE_CONFIG` in section 2 of the notebook:

```python
CELL_LINE_CONFIG["MyCellLine"] = {
    "filename":      "MyCellLine_ERG1_Report.xlsx",
    "erg1_pattern":  "MyCellLine_ERG1",   # appears in ERG1 column names
    "ctrl_pattern":  "MyCellLine_WT",     # appears in control column names
    "ctrl_exclude":  None,                # set to a substring if needed to disambiguate
    "drop_patterns": [],                  # columns to ignore entirely
}
```

Then add the new file to `REQUIRED_FILES` in section 1. The rest of the pipeline picks it up automatically.

---

## GlyGen reference CSVs

GlyGen serves data through a JavaScript portal, so direct download URLs are not stable and command-line tools like `wget` and `requests` return only the HTML shell.

### How to obtain the CSVs

1. Open https://data.glygen.org/GLY_000150 in a browser. Wait 5 to 10 seconds for the React app to load.
2. On the dataset page, click the link to `human_proteoform_glycosylation_sites_glyconnect.csv` in the Files section. The browser downloads the real CSV (tens of megabytes).
3. Repeat for https://data.glygen.org/GLY_000149 to obtain `human_proteoform_glycosylation_sites_unicarbkb.csv`.
4. Place both files in the working directory, in `./gala_pipeline/glygen/`, or in `./glygen/`.

Do not right-click on the URL to "Save link as", because that saves the HTML shell. Click the link inside the loaded page.

### Expected size

| File | Approximate size |
|---|---|
| `human_proteoform_glycosylation_sites_glyconnect.csv` | 20 to 100 MB |
| `human_proteoform_glycosylation_sites_unicarbkb.csv` | 5 to 50 MB |

If you see a file that is only a few kilobytes, it is the HTML error page; redownload.

### Expected schema

The notebook auto-detects columns by name from these candidate lists:

- UniProt accession column: `uniprotkb_canonical_ac`, `uniprotkb_ac`, `uniprot_canonical_ac`, `uniprot_canonical`, `uniprotkb`, `uniprot_acc`, `uniprot`
- Position column: `glycosylation_site_uniprotkb`, `glycosylation_site`, `site_uniprotkb`, `start_pos`, `position`, `site`
- Glycan type column: `saccharide_subtype`, `glycosylation_subtype`, `glycan_subtype`, `subtype`, `glycosylation_type`, `glycan_type`, `saccharide`
- Amino acid column (optional): `amino_acid`, `residue`, `site_aa`, `aa`
- Cross-reference columns: `xref_key`, `xref_id` (exact match)

Rows are kept only when:
- The glycan type matches `o-link`, `galnac` or `mucin` and does NOT match `o-glcnac`, `glcnac`, `mannose`, `fucose`, `glucose`, `n-link`, `n-glyc`.
- The amino acid is S or T (or unknown but resolvable from AlphaFold).
- The `xref_key` is `protein_xref_pubmed`, `protein_xref_glyconnect` or `protein_xref_unicarbkb_ds`.

If GlyGen changes the schema and a candidate column is renamed, edit the corresponding `*_CANDIDATES` list in section 6. The notebook will print which column it picked for each field so the mismatch is easy to spot.

---

## Output structure

After a successful run, the notebook produces a `gala_pipeline/` folder with the following layout:

```
gala_pipeline/
  cache/
    af_cache_human.pkl
  alphafold_pdb/
    AF-<accession>-F1-model_v6.pdb           # one per training protein
  uniprot_json/
    <accession>.json                         # one per training protein
  esmc_cache/
    esmc600_residue_embeddings_cache.pkl
  hf_models/                                  # ESM-C weights downloaded by HuggingFace
  glygen/                                     # GlyGen CSVs if placed here
  tables/
    training_glycoforms_topN.csv
    training_sites.csv
    training_matrices.pkl
    shap_feature_importance.csv
    ...
  models/
    cb_final_M1.pkl
    cb_final_M2.pkl
    cb_final_M3.pkl
    iso_M1.pkl, iso_M2.pkl, iso_M3.pkl
    scaler_esmc.pkl, pca_esmc.pkl
    oof_*.csv, fold_results_*.csv
    alt_models_comparison_*.csv
  plots/
    rsasa_distributions.{png,svg}
    benchmark_M2.{png,svg}, benchmark_M3.{png,svg}
    shap_M3.{png,svg}
    kpc47_validation.{png,svg}                # if KPC47 run
    kic_hmp_validation.{png,svg}              # if KIC vs HMP run
  kpc47_validation/                           # if KPC47 run
  kic_hmp_validation/                         # if KIC vs HMP run
```

Re-runs are incremental: AlphaFold PDBs, UniProt JSONs, FreeSASA results and ESM-C embeddings are cached. Deleting the `gala_pipeline/` folder triggers a full rebuild.