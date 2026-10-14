# Differential translation of PRICE ORFs

`run_price_joint_deseq2.R` takes the joint PRICE table produced by [`run_price_joint.sh`](README.md) and compares ORF translation between conditions. It normalizes the ORF counts with size factors computed from the gene counts of the same samples and then either:

- runs **DESeq2** when the design has replicates, or
- computes **descriptive log2 fold changes** when there are no replicates.

The output is the PRICE table with gene symbols, normalized counts for every sample, the ORF / gene ratio and the results of the comparison.

## Why normalize with the gene counts?

The read counts in the PRICE table are estimated but **not normalized**, so they cannot be compared directly between samples with different sequencing depths.

The script computes DESeq2 size factors (median of ratios) from the gene counts of each sample, with mitochondrial genes excluded. These counts cover all expressed genes and do not depend on PRICE's ORF calls or start-codon predictions. Non-canonical ORFs are deliberately **not** used for the normalization: if their translation changes between conditions for biological reasons, using them as a reference would remove exactly that signal.

DESeq2 receives the raw ORF counts (rounded) together with these size factors and applies the normalization inside its model. The ORF counts must never be normalized before they reach DESeq2.

## ORF / gene ratio

A change in an ORF can simply reflect a change of its whole gene: if a uORF goes from 10 to 40 reads while its gene goes from 1,000 to 4,000, the uORF has not changed specifically. To separate both effects, the script relates every ORF to the total reads of its gene.

**Gene total.** The gene counts loaded for the size factors, normalized with the same size factors. When PRICE merged several genes (`ENSG..._ENSG...`), their counts are summed. ORFs without a gene in the counts table (e.g. `orphan`) get `NA`.

**Ratio per sample.**

```
<sample>_ratio = <sample>_norm / <sample>_gene_norm
```

The share of the gene's reads that falls on the ORF. Because ORF counts come from PRICE and gene counts from the counting step, the two are not counted the same way, so the ratio can occasionally exceed 1. This does not affect the comparison between conditions, which uses the ratio's change.

**Ratio fold change per comparison.**

```
ratio_log2FC = log2FC(ORF) − log2FC(gene)      # log2 of (ORF fold change / gene fold change)
```

- `ratio_log2FC ≈ 0`: the ORF changes like its gene.
- `ratio_log2FC > 0`: the ORF increases more than its gene, i.e. a larger share of the gene's translation goes to the ORF.
- `ratio_log2FC < 0`: the ORF increases less (or decreases more) than its gene.

The size factors cancel in the ratio, so this measure does not depend on the normalization. Since the gene counts come from Ribo-seq, the gene total measures ribosomes on the whole gene (mRNA amount × translation); separating the mRNA level would require RNA-seq of the same samples.

## Input

| Input        | Location                                                                          |
| ------------ | --------------------------------------------------------------------------------- |
| PRICE table  | `${BASE_DIR}/<PROJID>/analysis/output/PRICE/joint_analysis/<PROJID>_<SPECIES>_joint.orfs.tsv` |
| Gene counts  | `${BASE_DIR}/<PROJID>/analysis/output/counts/all_unique/<sample>.gene_counts.tsv` |
| Sample sheet | Any path, given as an argument                                                    |

**Gene counts.** The first column must be the gene ID and the last column the counts. featureCounts output and simple two-column tables both work. Summary rows starting with `__` or `N_` are ignored.

**Sample names.** Each sample name must match:

- its gene counts file: `<sample>.gene_counts.tsv`
- its column in the PRICE table. PRICE names the columns after the full BAM path; the script reduces them to the file name and drops a trailing `_toGenome`. For example `.../toGenome/HT29_ctrl2_toGenome` becomes `HT29_ctrl2`.

## Usage

```bash
$BASE_DIR/scripts/scripts_andres/run_price_joint_deseq2.R <PROJID> <SPECIES> <SAMPLE_SHEET> <REFERENCE>
```

| Argument       | Description                                                   | Example                  |
| -------------- | ------------------------------------------------------------- | ------------------------ |
| `PROJID`       | Project ID; also used as the folder name                      | `48138_AE`               |
| `SPECIES`      | Reference genome: `hg19` or `mm10`                             | `hg19`                   |
| `SAMPLE_SHEET` | Tab-separated file describing the samples (see below)          | `samples_48138_AE.tsv`   |
| `REFERENCE`    | Value of the `condition` column used as baseline               | `ctrl`                   |

The script runs in less than a minute.

### Sample sheet

A tab-separated file with the columns `sample` and `condition`. Any additional column is used as a covariate.

- `condition` is the factor being compared. Every value is compared against `REFERENCE`, so `REFERENCE` must be a value of this column, not of a covariate.
- Covariates are adjusted for. With replicates, the DESeq2 design is `~ covariates + condition`. Without replicates, fold changes are computed separately within each level of the covariates.

To compare a different factor, move it to the `condition` column and turn the other one into a covariate.

### Example with replicates (DESeq2)

```
sample	condition
HT29_ctrl2	ctrl
HT29_ctrl3	ctrl
HT29_ctrl4	ctrl
HT29_iri2	iri
HT29_iri3	iri
HT29_iri4	iri
```

```bash
$BASE_DIR/scripts/scripts_andres/run_price_joint_deseq2.R 48138_AE hg19 samples_48138_AE.tsv ctrl
```

Output: `48138_AE_hg19_joint.iri_vs_ctrl.deseq2.tsv`

### Example without replicates (descriptive fold changes)

To compare gc7-treated with untreated cells, separately in proliferating and in senescent cells:

```
sample	condition	state
BJ_pro	ctrl	pro
BJ_pro_gc7	gc7	pro
BJ_sen	ctrl	sen
BJ_sen_gc7	gc7	sen
```

```bash
$BASE_DIR/scripts/scripts_andres/run_price_joint_deseq2.R 48138_AB hg19 AB_contrast_gc7.tsv ctrl
```

Output: `48138_AB_hg19_joint.gc7_vs_ctrl.log2fc.tsv`, with the comparisons BJ_pro_gc7 vs BJ_pro and BJ_sen_gc7 vs BJ_sen.

## Analysis modes

The script chooses the mode from the sample sheet:

| Sample sheet                                                              | Mode                    | Output suffix  |
| ------------------------------------------------------------------------- | ----------------------- | -------------- |
| At least one combination of condition and covariates appears more than once | DESeq2                  | `.deseq2.tsv`  |
| Every combination appears only once                                         | Descriptive log2 fold changes | `.log2fc.tsv`  |

**DESeq2 mode.** DESeq2 is run on the ORF counts with the size factors from the gene counts. Each condition is tested against the reference.

In both modes the gene fold change is computed the same way as the ORF fold change, so `ratio_log2FC` is directly the difference of the two columns. In DESeq2 mode the gene fold change comes from DESeq2 on the gene counts, with the same design and size factors. The ratio fold change is descriptive in both modes (no p value).

**Descriptive mode.** Without replicates DESeq2 cannot estimate the variability between samples of the same condition, so no p values can be computed. Instead, for each level of the covariates, the script computes:

```
log2FC = log2((normalized counts + 1) / (normalized counts of the reference + 1))
```

When there are exactly two levels of the covariates, it also reports the difference between the two fold changes. This shows whether the effect of the condition differs between them (for example, whether gc7 acts differently in senescent and in proliferating cells).

## What the script does

1. Reads the sample sheet, checks the reference and decides the analysis mode.
2. Reads the PRICE table and renames the sample columns to the sample names.
3. Reads the gene counts, excludes mitochondrial genes and computes DESeq2 size factors.
4. Corrects the PRICE p values with Benjamini-Hochberg and selects the translated ORFs (`price_padj < 0.05`). Only these ORFs are compared.
5. Runs DESeq2, or computes the descriptive fold changes.
6. Adds gene symbols, normalized ORF and gene counts, the ORF / gene ratio per sample and the gene and ratio fold changes, and writes the output files.
7. Prints a summary: size factors, number of translated ORFs, number of changed ORFs per comparison and number of non-canonical ORFs whose share of the gene changes (|ratio_log2FC| ≥ 1, at least 10 normalized reads in both groups).

The threshold for translated ORFs is set by `PRICE_FDR` at the top of the script.

## Output

Files are written to `${BASE_DIR}/<PROJID>/analysis/output/PRICE/joint_analysis/`:

| File                                             | Content                                         |
| ------------------------------------------------ | ----------------------------------------------- |
| `<PROJID>_<SPECIES>_joint.<comparison>.deseq2.tsv` | Results with replicates                         |
| `<PROJID>_<SPECIES>_joint.<comparison>.log2fc.tsv` | Results without replicates                      |
| `<PROJID>_<SPECIES>_joint.size_factors.tsv`        | Size factors per sample                         |

`<comparison>` is `<condition>_vs_<reference>`, for example `iri_vs_ctrl`. Different comparisons therefore write to different files.

### Columns of the results table

All rows of the PRICE table are kept. Comparison columns are `NA` for ORFs that were not called as translated.

| Column                         | Meaning                                                                         |
| ------------------------------ | ------------------------------------------------------------------------------- |
| `Gene`, `Id`, `Location`, ...  | Original PRICE columns                                                          |
| `gene_symbol`                  | Gene symbol (see notes below)                                                   |
| `price_padj`                   | PRICE p value corrected with Benjamini-Hochberg                                 |
| `<sample>`                     | Raw PRICE counts                                                                |
| `<sample>_norm`                | Counts divided by the size factor of the sample                                 |
| `<sample>_gene_norm`           | Normalized gene counts of the ORF's gene                                        |
| `<sample>_ratio`               | `<sample>_norm / <sample>_gene_norm`                                             |

DESeq2 mode adds, for each comparison:

| Column                                     | Meaning                                    |
| ------------------------------------------ | ------------------------------------------ |
| `baseMean`                                 | Mean of the normalized counts              |
| `<cond>_vs_<ref>_log2FoldChange`           | log2 fold change                           |
| `<cond>_vs_<ref>_lfcSE`                    | Standard error of the log2 fold change     |
| `<cond>_vs_<ref>_stat`                     | Wald statistic                             |
| `<cond>_vs_<ref>_pvalue`, `_padj`          | p value and Benjamini-Hochberg adjusted p value |
| `<cond>_vs_<ref>_gene_log2FC`              | log2 fold change of the ORF's gene (DESeq2 on the gene counts) |
| `<cond>_vs_<ref>_ratio_log2FC`             | `log2FoldChange − gene_log2FC`             |

Descriptive mode adds, for each comparison:

| Column                                     | Meaning                                                     |
| ------------------------------------------ | ----------------------------------------------------------- |
| `<cond>_vs_<ref>_in_<level>_log2FC`        | log2 fold change within one level of the covariates        |
| `<cond>_vs_<ref>_in_<level>_gene_log2FC`   | log2 fold change of the ORF's gene, same level              |
| `<cond>_vs_<ref>_in_<level>_ratio_log2FC`  | `log2FC − gene_log2FC`                                       |
| `<cond>_vs_<ref>_<level2>_minus_<level1>`  | Difference between the two fold changes (only with two levels) |

### Size factors file

| Column                       | Meaning                                                             |
| ---------------------------- | ------------------------------------------------------------------- |
| `size_factor_gene_counts`    | Size factors used in the analysis                                   |
| `size_factor_canonical_orfs` | Size factors from the canonical translated ORFs (`CDS`, `Ext`, `Trunc`, `Variant`; no chrM) |

The second column is a sanity check. Similar values mean that the normalization does not depend on the reference chosen. Clear differences between conditions suggest a global shift that is worth looking into.

## Notes for interpretation

- **Fold changes without replicates are exploratory.** Use them to prioritize candidates, not as final results. ORFs with few reads give very noisy fold changes; the summary counts only ORFs with at least 10 normalized reads in both groups and |log2FC| ≥ 1.
- **Normalization assumes that canonical translation does not change globally.** If a condition reduces translation overall, no normalization based on the samples themselves can detect it; that would require spike-ins. The results describe translation of each ORF relative to the bulk of translation.
- **Isoforms share reads.** PRICE reports ORFs from different isoforms as separate rows, and reads in shared exons are counted in each. The same change can therefore appear in several rows of one gene.
- **Gene symbols** come from `org.Hs.eg.db` or `org.Mm.eg.db`, which use NCBI nomenclature: mitochondrial genes appear as `COX1`, `ND1`, etc. IDs without a symbol keep their Ensembl ID. Genes that PRICE merged are joined with `_`, as in the `Gene` column.

## Troubleshooting

| Problem                                              | Fix                                                                                 |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `Reference '...' is not a value of the 'condition' column` | The reference must be a value of `condition`, not of a covariate                |
| `Samples not found in the PRICE table`               | Make the sample names match the PRICE columns; the error lists the names found      |
| `Gene counts not found`                              | Check the name of the file `<sample>.gene_counts.tsv`                               |
| `Package '...' is not installed`                     | Install it with the `BiocManager::install()` command shown in the error             |
| `The design ... leaves no residual degrees of freedom` | Use fewer covariates                                                              |
| `BASE_DIR is not defined`                            | Define `BASE_DIR` in your `~/.bashrc`                                               |

## References

- Love MI, Huber W, Anders S. Moderated estimation of fold change and dispersion for RNA-seq data with DESeq2. *Genome Biology* 15, 550 (2014).
- PRICE documentation: <https://github.com/erhard-lab/gedi/wiki/Price>
