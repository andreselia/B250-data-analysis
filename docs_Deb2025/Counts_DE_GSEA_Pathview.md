# Get gene counts with gene symbols
This script translates ENST to Gene Symbols and generates a table with the read count for each gene

```
bsub -q medium -R "rusage[mem=30G]" $BASE_DIR/software/counts/get_counts_symbols.sh 47681 all_unique hg19
```
Output will look like this:

Processing sample: A549_BPTES

Processing sample: A549_CB839

Processing sample: A549_Ctrl

........

------------------------------------------------------------

Done. Output files:

  /omics/groups/OE0532/internal/....//47681/analysis/output/counts/all_unique/A549_BPTES.gene_counts.tsv

  /omics/groups/OE0532/internal/....//47681/analysis/output/counts/all_unique/A549_CB839.gene_counts.tsv

.........

Transcript-to-gene map used:

  /omics/groups/OE0532/internal/....//47681/analysis/output/counts/all_unique/transcript_to_gene.hg19.tsv


## Diff expr analysis

### What it does

1. Restricts analysis to the samples listed in a contrast table -- other
   samples in the counts folder are ignored.
2. Builds a raw counts matrix from `*.gene_counts.tsv` files.
3. Runs `DESeq2` (`~group`, low-count filter `rowSums >= 10`) and exports
   **every pairwise comparison** between groups (e.g. `Ctrl`/`2hr`/`24hr` ->
   `2hr_vs_Ctrl`, `24hr_vs_Ctrl`, `24hr_vs_2hr`).
4. Exports VST-normalized counts (all genes) and linear normalized counts.
5. For a user-supplied gene list: boxplot, row-z-score heatmap, and two QC
   histograms (all genes vs. selected genes, same x-axis) -- all covering
   every sample in the contrast table.

**No replicates?** If any group has <2 samples, no statistical test is
possible (zero residual degrees of freedom, true for any method). The
script detects this automatically, skips `DESeq()`, and instead exports
plain point-estimate log2FC (no p-value) plus VST in blind mode -- all
those filenames get a `_NO_STATS_blindVST` / `NO_PVALUE` suffix so they're
never confused with a real statistical result.

### Contrast table format

```bash
vim $BASE_DIR/47681/analysis/input/metadata/WP_contrast.tsv
```

```
sampleid	group
HEK293_Ctrl_1	Ctrl
HEK293_2hr_1	2hr
HEK293_24hr_1	24hr
HEK293_Ctrl_2	Ctrl
HEK293_2hr_2 	2hr
HEK293_24hr_2	24hr
```

`sampleid` must match `<sampleid>.gene_counts.tsv` in the counts folder.

**Reference group / comparison direction**: alphabetical sorting would put
`Ctrl` after `24hr`, flipping the intended direction. Fixed by, in order:
1. an optional `order` (or `priority`) column -- lowest value = reference
   in every comparison it's part of (best for >2 groups);
2. auto-detected `ctrl`/`control` group name;
3. row order of first appearance (with a warning in the log).

```
sampleid        group   order
HEK293_Ctrl_2   Ctrl    1
HEK293_2hr_2    2hr     2
HEK293_24hr_2   24hr    3
```

### Gene list format

```bash
vim $BASE_DIR/47681/analysis/input/metadata/WP_contrast_genelist.tsv
```

```
MT-CO1
MT-ND1
MT-ATP6
GAPDH
```

### Usage

```bash
bsub -q medium -R "rusage[mem=30G]" \
  $BASE_DIR/software/counts/run_deseq2.sh \
  <project_id> <bam_type> <contrast_table_name> <gene_list_path> [genome]
```

| Argument | Description | Default |
|---|---|---|
| `project_id` | Project folder under `$BASE_DIR` | required |
| `bam_type` | `all`, `all_unique`, or `hq_unique` | required |
| `contrast_table_name` | Name (no path/extension) under `analysis/input/metadata/` | `deseq2_contrast_table` |
| `gene_list_path` | Full path to gene list file | required |
| `genome` | Logging only | `hg19` |

```bash
bsub -q medium -R "rusage[mem=30G]" $BASE_DIR/software/counts/run_deseq2.sh \
  47681 all_unique WP_contrast \
  $BASE_DIR/47681/analysis/input/metadata/WP_contrast_genelist.tsv
```

### Output

```
$BASE_DIR/<project_id>/analysis/output/counts/<bam_type>/deseq2_<contrast_table_name>/
├── raw_counts_matrix_<contrast_table_name>.tsv
├── results_<contrast_table_name>_<groupB>_vs_<groupA>.tsv       # or log2FC_only_NO_PVALUE_* if no replicates
├── normalized_transformed_counts_<contrast_table_name>.tsv      # VST, all genes
├── normalized_counts_linear_<contrast_table_name>.tsv
├── qc_histogram_all_genes_<contrast_table_name>.pdf
├── qc_histogram_selected_genes_<contrast_table_name>.pdf
├── selected_genes_normalized_transformed_counts_<contrast_table_name>.tsv
├── selected_genes_plot_<contrast_table_name>.pdf                # boxplot
└── selected_genes_heatmap_<contrast_table_name>.pdf              # row z-score
```

(Without replicates, filenames carry `_NO_STATS_blindVST` instead of the
plain contrast table name.) All paths are printed in the `bsub` job log.

### Requirements

R with `DESeq2`, `ggplot2`, `pheatmap` (`module load R/4.4.3-GCCcore-14.1.0`).


## KEGG pathway visualization
`pathway_plot.r` / `run_pathway_plot.sh` 

### What it does

Overlays your per-gene log2 fold changes directly onto the official KEGG
pathway diagram, coloring each enzyme node **red** (up) or **blue** (down)
relative to zero. It reuses the fold-change tables already produced by
`deseq2_analysis.r` -- no data is recomputed.

For a given comparison, it looks for, in order:

1. `results_<contrast_table_name>_<groupB_vs_groupA>.tsv` (produced when you
   had replicates -- uses the `log2FoldChange` column).
2. `log2FC_only_NO_PVALUE_<contrast_table_name>_<groupB_vs_groupA>.tsv`
   (produced when you had no replicates -- uses the `log2FC` column).

Whichever exists is used automatically; you don't need to specify which one.

By default it plots a curated set of **amino acid metabolism/biosynthesis**
KEGG pathways:

| KEGG ID | Pathway |
|---|---|
| 00250 | Alanine, aspartate and glutamate metabolism |
| 00260 | Glycine, serine and threonine metabolism |
| 00270 | Cysteine and methionine metabolism |
| 00280 | Valine, leucine and isoleucine degradation |
| 00290 | Valine, leucine and isoleucine biosynthesis |
| 00220 | Arginine biosynthesis |
| 00330 | Arginine and proline metabolism |
| 00340 | Histidine metabolism |
| 00350 | Tyrosine metabolism |
| 00360 | Phenylalanine metabolism |
| 00380 | Tryptophan metabolism |
| 00400 | Phenylalanine, tyrosine and tryptophan biosynthesis |
| 00410 | beta-Alanine metabolism |
| 00300 | Lysine biosynthesis |
| 00310 | Lysine degradation |
| 00970 | Aminoacyl-tRNA biosynthesis |

You can override this list with any comma-separated KEGG pathway IDs.

### Requirements

Bioconductor packages, installed once:

```r
if (!requireNamespace("BiocManager", quietly = TRUE)) install.packages("BiocManager")
BiocManager::install(c("pathview", "org.Hs.eg.db"))
# for mouse:
BiocManager::install("org.Mm.eg.db")
```

`pathview` needs internet access the first time it plots a given pathway
(it downloads and locally caches the KEGG reference diagram).

### Usage

```bash
bsub -q medium -R "rusage[mem=15G]" \
  $BASE_DIR/software/counts/run_pathway_plot.sh \
  <project_id> <bam_type> <contrast_table_name> <groupB_vs_groupA> [pathway_ids] [species]
```

| Argument | Description | Default |
|---|---|---|
| `project_id` | Project folder name under `$BASE_DIR` | required |
| `bam_type` | Must match the `bam_type` used upstream | required |
| `contrast_table_name` | Same contrast table name used in `run_deseq2.sh` | required |
| `groupB_vs_groupA` | Comparison label exactly as it appears in the `deseq2_analysis.r` output filenames, e.g. `24hr_vs_Ctrl` | required |
| `pathway_ids` | Comma-separated KEGG pathway IDs (no species prefix) | amino acid pathway list above |
| `species` | KEGG species code | `hsa` (human); use `mmu` for mouse |

Examples:

```bash
# All default amino acid pathways
bsub -q medium -R "rusage[mem=15G]" $BASE_DIR/software/counts/run_pathway_plot.sh \
  47681 all_unique WP_contrast 24hr_vs_Ctrl

# Just arginine/proline metabolism
bsub -q medium -R "rusage[mem=15G]" $BASE_DIR/software/counts/run_pathway_plot.sh \
  47681 all_unique WP_contrast 24hr_vs_Ctrl 00330
```

### Output

```
$BASE_DIR/<project_id>/analysis/output/counts/<bam_type>/deseq2_<contrast_table_name>/pathview_<groupB_vs_groupA>/
├── hsa00250.<groupB_vs_groupA>.png   # colored pathway diagram
├── hsa00330.<groupB_vs_groupA>.png
└── ...
```

## Gene Set Enrichment Analysis (KEGG)
`gsea_analysis.r` / `run_gsea.sh` -- 

### What it does

Runs preranked GSEA (via `fgsea`) against the full KEGG canonical pathway
collection (`msigdbr`, category `C2` / `CP:KEGG`). It reuses the fold-change
tables already produced by `deseq2_analysis.r` -- no data is recomputed.

For a given comparison, it looks for, in order:

1. `results_<contrast_table_name>_<groupB_vs_groupA>.tsv` (produced when you
   had replicates) -- ranks genes by the **Wald statistic** (`stat` column),
   the recommended ranking metric for preranked GSEA since it combines
   direction and statistical confidence.
2. `log2FC_only_NO_PVALUE_<contrast_table_name>_<groupB_vs_groupA>.tsv`
   (produced when you had no replicates) -- ranks genes by plain `log2FC` as
   a fallback. A warning is printed: without proper statistical weighting,
   the resulting p-values/NES should be interpreted as exploratory only.

### Two run modes

| `filter` argument | Behavior |
|---|---|
| omitted / empty (default) | Only the **total** KEGG run (all ~186 canonical pathways tested) |
| `AA` | Also runs a **second**, restricted GSEA against only the KEGG pathways whose name matches an amino-acid-related keyword (built-in list: the 20 amino acids + `AMINO_ACID` + `AMINOACYL_TRNA`) |
| any other string | Treated as comma-separated **custom keyword(s)** for the restricted run, e.g. `METABOLISM` for a much broader metabolic filter, or `GLUTAMINE,ASPARTATE` for a narrower custom subset |

### Requirements

```r
install.packages(c("msigdbr", "fgsea"))
```

### Usage

```bash
bsub -q medium -R "rusage[mem=15G]" \
  $BASE_DIR/software/counts/run_gsea.sh \
  <project_id> <bam_type> <contrast_table_name> <groupB_vs_groupA> [filter] [species]
```

| Argument | Description | Default |
|---|---|---|
| `project_id` | Project folder name under `$BASE_DIR` | required |
| `bam_type` | Must match the `bam_type` used upstream | required |
| `contrast_table_name` | Same contrast table name used in `run_deseq2.sh` | required |
| `groupB_vs_groupA` | Comparison label exactly as it appears in the `deseq2_analysis.r` output filenames, e.g. `24hr_vs_Ctrl` | required |
| `filter` | `""` (total only), `AA` (+ amino acid filter), or custom keyword(s) | `""` |
| `species` | `msigdbr` species name | `Homo sapiens` (use `Mus musculus` for mouse) |

Examples:

```bash
# Default: total KEGG only
bsub -q medium -R "rusage[mem=15G]" $BASE_DIR/software/counts/run_gsea.sh \
  47681 all_unique WP_contrast 24hr_vs_Ctrl

# Total KEGG + amino acid metabolism filter
bsub -q medium -R "rusage[mem=15G]" $BASE_DIR/software/counts/run_gsea.sh \
  47681 all_unique WP_contrast 24hr_vs_Ctrl AA

# Total KEGG + broader metabolism filter (custom keyword)
bsub -q medium -R "rusage[mem=15G]" $BASE_DIR/software/counts/run_gsea.sh \
  47681 all_unique WP_contrast 24hr_vs_Ctrl METABOLISM
```

### Output

```
$BASE_DIR/<project_id>/analysis/output/counts/<bam_type>/deseq2_<contrast_table_name>/gsea_<groupB_vs_groupA>/
├── gsea_KEGG_total_<contrast_table_name>_<groupB_vs_groupA>.tsv          # all ~186 pathways, sorted by p-value
├── gsea_KEGG_total_top20_<contrast_table_name>_<groupB_vs_groupA>.pdf    # NES barplot, top 20
├── gsea_aminoacid_metabolism_<contrast_table_name>_<groupB_vs_groupA>.tsv   # only if filter requested
├── gsea_aminoacid_metabolism_<contrast_table_name>_<groupB_vs_groupA>.pdf  # NES barplot, all matched sets
└── gsea_enrichment_curve_top_aminoacid_<contrast_table_name>_<groupB_vs_groupA>.pdf  # classic GSEA curve for the top hit in the filtered set
```

The `*.tsv` results include standard `fgsea` columns: `pathway`, `pval`,
`padj`, `NES`, `ES`, `size`, and `leadingEdge` (semicolon-separated list of
the genes driving the enrichment).
