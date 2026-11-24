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

> **Counts are whole-gene, UTRs included.** For RNA-seq this is the usual
> thing to want. For **ribosome profiling** it means 5'UTR reads (uORFs,
> upstream initiation) are pooled into the gene total, so a gene whose
> regulation happens through a uORF can look unchanged — or change in the
> *opposite* direction to its CDS. If that is the question, produce
> CDS-restricted counts here, upstream; nothing downstream can correct for
> it.


## Diff expr analysis

### What it does

1. Restricts analysis to the samples listed in a contrast table -- other
   samples in the counts folder are ignored.
2. Builds a raw counts matrix from `*.gene_counts.tsv` files.
3. Runs `DESeq2` (`~group`, low-count filter `rowSums >= 10`) and exports
   **every pairwise comparison** between groups (e.g. `Ctrl`/`2hr`/`24hr` ->
   `2hr_vs_Ctrl`, `24hr_vs_Ctrl`, `24hr_vs_2hr`). Optional columns of the
   contrast table add a **batch** correction (`~batch + group`) or **split the
   comparisons** by another variable (e.g. `state`), including a test of
   whether the effect differs between its levels (interaction).
4. Saves the fitted `DESeq2` object as `dds_<contrast_table_name>.rds`.
5. Exports VST-normalized counts (all genes) and linear normalized counts.
6. Draws a **PCA** of the samples (and, with a batch column, a second PCA with
   the batch effect removed).
7. For a user-supplied gene list: boxplot, row-z-score heatmap, and two QC
   histograms (all genes vs. selected genes, same x-axis) -- all covering
   every sample in the contrast table.

**No replicates?** If any group has <2 samples, no statistical test is
possible (zero residual degrees of freedom, true for any method). The
script detects this automatically, skips `DESeq()`, and instead exports
plain point-estimate log2FC (no p-value) plus VST in blind mode -- all
those filenames get a `_NO_STATS_blindVST` / `NO_PVALUE` suffix so 
never confused with a real statistical result. With a split variable
(e.g. `state`), "group" means every combination of that variable and
`group`, and the log2FC are computed within each level (plus their
difference when there are two levels).

**Only 2 replicates?** Testing works, but the script prints a note: per-gene
dispersion is then largely borrowed from the fitted trend and genome-wide
`padj` is conservative. Prefer `log2FC_shrunk` for ranking and for any
fold-change-colored figure, and see `padj_genelist` below.

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

Leading/trailing spaces in any field are removed, so a stray space after a
`sampleid` no longer breaks the lookup of its counts file.

**Optional columns**

| Column | Effect |
|---|---|
| `order` / `priority` | Comparison direction (see above) |
| `batch` | Only adjusts the model: design `~batch + group`. Use it for library preparation batches, sequencing runs, or paired replicates (e.g. `Ctrl_1`/`2hr_1` processed together). Each batch must contain more than one group, otherwise the script stops. Ignored without replicates. |
| any other column (e.g. `state`) | **Splits the comparisons**: groups are compared within each level of that column. With exactly two levels, the difference between the two effects is also tested (interaction). |

Example: effect of gc7 in proliferating and in senescent cells, and whether
the two effects differ (two replicates per combination, two batches):

```
sampleid        group   state   batch
BJ_pro          Ctrl    pro     rep2
BJ_pro_gc7      gc7     pro     rep2
BJ_sen          Ctrl    sen     rep2
BJ_sen_gc7      gc7     sen     rep2
BJ_pro1         Ctrl    pro     rep1
BJ_pro1_gc7     gc7     pro     rep1
BJ_sen1         Ctrl    sen     rep1
BJ_sen1_gc7     gc7     sen     rep1
```

Every combination of `state` and `group` becomes one level of the model
(`pro_Ctrl`, `pro_gc7`, `sen_Ctrl`, `sen_gc7`), the design is
`~batch + combo`, and all eight samples are fitted together. Comparisons:

| Comparison label | Question |
|---|---|
| `gc7_vs_Ctrl_in_pro` | Effect of gc7 in proliferating cells |
| `gc7_vs_Ctrl_in_sen` | Effect of gc7 in senescent cells |
| `gc7_vs_Ctrl_sen_minus_pro` | Is the effect of gc7 different in senescent cells? (interaction: `log2FoldChange` = difference between the two effects) |

A non-significant interaction does not show that the two effects are equal;
interaction tests need more replicates than simple comparisons.

> **The reference level of the split variable comes from ROW ORDER, not from
> the `order` column.** `order` only sets the direction of `group`. The first
> level of the split variable to appear in the table becomes the baseline,
> which is why the label above reads `sen_minus_pro` and not the reverse.
> Reordering rows silently flips the sign of every interaction. The script
> now prints the chosen reference level and spells out how to read each
> interaction label — **check the log before interpreting signs.**

**Getting the other main effect.** With a split variable the script compares
`group` levels *within* each stratum; it does not compare the strata to each
other. To get that effect too, write a second contrast table with the roles
swapped (what was `group` becomes `state` and vice versa). It fits exactly
the same four cells, so this is not double testing — it just reports the
other axis. The interaction comes out identical in magnitude.

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

One gene symbol per line. Ignored: blank lines, whole-line comments
(`# ...`), and **trailing inline comments after a symbol** — so an annotated
list works directly:

```
# --- Cytoplasmic aminoacyl-tRNA synthetases ---
AARS      # Ala   (current: AARS1)
RARS      # Arg   (current: RARS1)
```

Duplicate symbols are collapsed, and the count of unique symbols is printed.

> Earlier versions only skipped lines *starting* with `#`. Annotated lists
> like the one above therefore matched nothing and silently produced empty or
> truncated heatmaps. This never affected any DE statistic (see the note
> below), only the gene-list plots.

**Symbols must match the annotation build.** Many symbols were renamed in
2018 while older builds (hg19 / GENCODE v19) keep the old ones — `RACK1` is
`GNB2L1`, `CCN1`/`CCN2` are `CYR61`/`CTGF`, and so on. Check a list before
running:

```bash
cut -f2 $BASE_DIR/<project_id>/analysis/output/counts/<bam_type>/transcript_to_gene.hg19.tsv \
  | sort -u > /tmp/hg19_symbols.txt
sed 's/#.*//' my_gene_list.txt | tr -d ' \t' | grep -v '^$' \
  | grep -vxFf /tmp/hg19_symbols.txt
```

Anything printed is a symbol that does not exist in that annotation.

**The gene list does not affect any DE statistic.** It only drives the
boxplot, the heatmap, the `selected_genes_*` tables and the `padj_genelist`
column. The `results_*.tsv` fold changes, p-values and genome-wide `padj` are
identical whatever list you pass, so there is no need to re-run the pipeline
to look at a different set of genes — subset
`normalized_transformed_counts_*.tsv` yourself, or re-run only if you want
the plots.

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
├── dds_<contrast_table_name>.rds                                # fitted DESeq2 object
├── results_<contrast_table_name>_<comparison>.tsv               # or log2FC_only_NO_PVALUE_* if no replicates
├── normalized_transformed_counts_<contrast_table_name>.tsv      # VST, all genes
├── normalized_counts_linear_<contrast_table_name>.tsv
├── pca_<contrast_table_name>.pdf / .tsv                         # PCA plot + PC coordinates
├── pca_batch_corrected_<contrast_table_name>.pdf / .tsv         # only with a batch column
├── qc_histogram_all_genes_<contrast_table_name>.pdf
├── qc_histogram_selected_genes_<contrast_table_name>.pdf
├── selected_genes_normalized_transformed_counts_<contrast_table_name>.tsv
├── selected_genes_plot_<contrast_table_name>.pdf                # boxplot
└── selected_genes_heatmap_<contrast_table_name>.pdf              # row z-score
```

`<comparison>` is `<groupB>_vs_<groupA>`; with a split variable,
`<groupB>_vs_<groupA>_in_<level>` and `<groupB>_vs_<groupA>_<level2>_minus_<level1>`.
(Without replicates, filenames carry `_NO_STATS_blindVST` instead of the
plain contrast table name.) All paths, and the number of genes with
`padj < 0.05` per comparison, are printed in the `bsub` job log.

#### Columns of `results_*.tsv`

Standard `DESeq2` columns (`baseMean`, `log2FoldChange`, `lfcSE`, `stat`,
`pvalue`, `padj`) plus:

| Column | Meaning |
|---|---|
| `log2FC_shrunk` | Shrunken fold change (`lfcShrink`, `ashr`). Low-count genes get large, unstable `log2FoldChange` values that dominate fold-change-ranked lists and fold-change-colored diagrams; shrinkage pulls them toward zero in proportion to their uncertainty. Only present if `ashr` is installed. |
| `lfcSE_shrunk` | Standard error of the shrunken estimate. |
| `padj_genelist` | BH correction computed **over the supplied gene list only**. `NA` for every gene outside it. |

`ashr` is used rather than `apeglm` because `apeglm` cannot handle arbitrary
numeric contrasts, which is exactly what the interaction terms are.

`log2FoldChange` and `stat` keep their original unshrunken values — the new
columns are additions, never replacements. This matters: `gsea_analysis.r`
ranks on `stat`, which `ashr` does not return, so overwriting these would
break the GSEA step silently. If you compute your own shrinkage downstream,
add it as a column; do not overwrite.

**When to report `padj_genelist`.** Genome-wide BH over ~15–20k genes is the
right default for a discovery screen, but it is the wrong denominator for a
panel specified *before* seeing the data: those genes are a pre-registered
hypothesis, not a screen. With few replicates the difference in power is
large. It is only defensible if the list really was fixed in advance — if you
add genes *because* you saw them come up, go back to the genome-wide `padj`,
and say in the methods which one you used and why.

#### Missing genes in the log

Symbols from the gene list that do not make it into the plots are reported in
two separate categories, because they mean completely different things:

- **Absent from the annotation entirely** — wrong or outdated symbol. Fix it.
  Reported right after the counts matrix is built.
- **Present but removed by the low-count filter** (`rowSums < 10`) — the gene
  is genuinely not expressed in these samples. That is a real result, not an
  error. Reported with the gene-list plots.

#### Reusing the fitted object

`dds_<contrast_table_name>.rds` lets you add contrasts, try other shrinkage
estimators, or redraw plots without re-running the cluster job:

```r
library(DESeq2)
dds <- readRDS("dds_WP_contrast.rds")
resultsNames(dds)                      # available coefficients
res <- results(dds, contrast = c("group", "24hr", "Ctrl"))
```

With a split variable, contrasts are numeric vectors over the `combo`
coefficients rather than `c("group", ...)` triples — read `resultsNames(dds)`
first.

**PCA.** Computed on the VST counts of the 500 most variable genes (as
`DESeq2::plotPCA`). Points are colored by `group`, shaped by the split
variable (or by `batch`), and labeled with the `sampleid`. With a `batch`
column, `pca_batch_corrected_*` shows the same PCA after removing the batch
effect with `limma::removeBatchEffect` (for visualization only; the
statistics use the batch term in the DESeq2 model). The `.tsv` files contain
the coordinates of the first principal components.

### Requirements

R with `DESeq2`, `ggplot2`, `pheatmap` (`module load R/4.4.3-GCCcore-14.1.0`).

Optional:
- `limma` — batch-corrected PCA.
- `ashr` — `log2FC_shrunk` / `lfcSE_shrunk` columns (`install.packages("ashr")`).

Both are detected at runtime: if missing, the script prints a note and
continues without that output.


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

> Note that it colors by the **unshrunken** `log2FoldChange`. With few
> replicates, low-count genes can show up as intensely colored nodes on
> essentially no evidence. Cross-check any striking node against `baseMean`,
> `padj` and `log2FC_shrunk` in the results table before reading anything
> into it.

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
| `groupB_vs_groupA` | Comparison label exactly as it appears in the `deseq2_analysis.r` output filenames, e.g. `24hr_vs_Ctrl`, or with a split variable `gc7_vs_Ctrl_in_pro` / `gc7_vs_Ctrl_sen_minus_pro` | required |
| `pathway_ids` | Comma-separated KEGG pathway IDs (no species prefix) | amino acid pathway list above |
| `species` | KEGG species code | `hsa` (human); use `mmu` for mouse |

> `bam_type` and `contrast_table_name` must match the `run_deseq2.sh` call
> exactly, and `groupB_vs_groupA` must match the generated filename character
> for character. Check the DESeq2 job log for the exact comparison labels
> before launching — a mismatch just means no input file is found.

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

## Gene Set Enrichment Analysis
`gsea_analysis.r` / `run_gsea.sh`

### What it does

Runs preranked GSEA (via `fgsea`) against one or more MSigDB collections. It
reuses the fold-change tables already produced by `deseq2_analysis.r` -- no
data is recomputed.

For a given comparison, it looks for, in order:

1. `results_<contrast_table_name>_<groupB_vs_groupA>.tsv` (produced when you
   had replicates) -- ranks genes by the **Wald statistic** (`stat` column),
   the recommended ranking metric for preranked GSEA since it combines
   direction and statistical confidence.
2. `log2FC_only_NO_PVALUE_<contrast_table_name>_<groupB_vs_groupA>.tsv`
   (produced when you had no replicates) -- ranks genes by plain `log2FC` as
   a fallback. A warning is printed: without proper statistical weighting,
   the resulting p-values/NES should be interpreted as exploratory only.

> Because this step depends on the `stat` column, never overwrite it with
> shrunken values in `results_*.tsv` -- `ashr` and `apeglm` do not return a
> `stat`, and the script now stops with an explicit error if it is missing.

Before testing, the ranked list is cleaned: `NA` values dropped, duplicated
gene symbols collapsed to the entry with the largest absolute statistic, and
the fraction of tied values reported (`fgsea` handles ties, but a heavily tied
ranking gives a less stable NES). The cleaned ranking is also written out as
`ranked_genes_*.rnk`, so it can be reused in GSEA desktop or tested against
any other gene set file without re-running anything.

### Collections

The 7th argument is a comma-separated list of MSigDB collections. Default is
`KEGG`, so existing commands keep their old behaviour.

| Name | MSigDB | Notes |
|---|---|---|
| `HALLMARK` | H | ~50 curated sets, low redundancy, the best general default |
| `KEGG` | C2 `CP:KEGG` | Metabolic and signalling pathways |
| `REACTOME` | C2 `CP:REACTOME` | Large, finely curated pathway tree |
| `WIKIPATHWAYS` | C2 `CP:WIKIPATHWAYS` | |
| `BIOCARTA` | C2 `CP:BIOCARTA` | |
| `PID` | C2 `CP:PID` | |
| `CP` | C2 `CP:*` | Every canonical pathway source at once |
| `GOBP` | C5 `GO:BP` | Biological process |
| `GOCC` | C5 `GO:CC` | Cellular component |
| `GOMF` | C5 `GO:MF` | Molecular function |
| `GO` | C5 `GO:*` | All three GO branches |
| `TFT` | C3 `TFT:*` | Transcription factor targets |
| `ONCOGENIC` | C6 | Oncogenic signatures |

Each collection gets its own results table, NES barplot and enrichment curve,
and **its own multiple-testing correction**. This is deliberate: `padj` inside
REACTOME is not comparable to `padj` inside HALLMARK, because the collections
differ enormously in size and in internal redundancy. Compare collections by
NES and by which genes drive the leading edge, not by `padj`.

`minSize` is 15 for the GO branches and 10 elsewhere; GO carries thousands of
small, highly overlapping sets and the lower floor makes the output unreadable
without changing which large terms come out.

### The optional keyword filter

The 5th argument adds a **second, restricted run inside every requested
collection**, keeping only gene sets whose name matches a keyword.

| `filter` argument | Behavior |
|---|---|
| omitted / empty (default) | Only the full run per collection |
| `AA` | Built-in amino acid list: the 20 amino acids + `AMINO_ACID` + `AMINOACYL_TRNA` |
| `PA` | Built-in polyamine list: `POLYAMINE`, `SPERMIDINE`, `SPERMINE`, `PUTRESCINE`, `ORNITHINE`, `ARGININE_AND_PROLINE`, `HYPUSINE`, translation elongation/initiation, `METHIONINE` |
| any other string | Comma-separated custom keyword(s), e.g. `METABOLISM`, or `GLUTAMINE,ASPARTATE` |

### Requirements

```r
install.packages(c("msigdbr", "fgsea"))
```

The script is written against both the old and the new `msigdbr` API and
detects which one is installed at runtime. Version 10.0 renamed the arguments
`category` / `subcategory` to `collection` / `subcollection`, and the returned
columns `gs_cat` / `gs_subcat` to `gs_collection` / `gs_subcollection`; a
script hard-coded to either spelling breaks on the other.

It also handles the KEGG rename. MSigDB 2023.1 renamed `CP:KEGG` to
`CP:KEGG_LEGACY` and added `CP:KEGG_MEDICUS`, which is a **different
curation**, not an update -- combining them double-counts overlapping
pathways. The script reports which KEGG subcollections your installation
offers and picks legacy first, matching what earlier runs of this pipeline
used. The choice is printed in the job log.

### Usage

```bash
bsub -q medium -R "rusage[mem=15G]" \
  $BASE_DIR/software/counts/run_gsea.sh \
  <project_id> <bam_type> <contrast_table_name> <groupB_vs_groupA> [filter] [species] [collections]
```

| Argument | Description | Default |
|---|---|---|
| `project_id` | Project folder name under `$BASE_DIR` | required |
| `bam_type` | Must match the `bam_type` used upstream | required |
| `contrast_table_name` | Same contrast table name used in `run_deseq2.sh` | required |
| `groupB_vs_groupA` | Comparison label exactly as it appears in the `deseq2_analysis.r` output filenames, e.g. `24hr_vs_Ctrl`, or with a split variable `gc7_vs_Ctrl_in_pro` / `gc7_vs_Ctrl_sen_minus_pro` | required |
| `filter` | `""` (no restricted run), `AA`, `PA`, or custom keyword(s) | `""` |
| `species` | `msigdbr` species name | `Homo sapiens` |
| `collections` | Comma-separated collection names from the table above | `KEGG` |

Positional arguments cannot be skipped. To reach `collections` you must pass
`filter` and `species` explicitly, using `""` for an empty filter.

Examples:

```bash
# Default: KEGG only, no filter (unchanged behaviour)
bsub -q medium -R "rusage[mem=15G]" $BASE_DIR/software/counts/run_gsea.sh \
  47681 all_unique WP_contrast 24hr_vs_Ctrl

# Four collections at once
bsub -q medium -R "rusage[mem=15G]" $BASE_DIR/software/counts/run_gsea.sh \
  47681 all_unique WP_contrast 24hr_vs_Ctrl "" "Homo sapiens" "HALLMARK,KEGG,REACTOME,GOBP"

# Hallmark + Reactome, with the polyamine keyword filter applied inside each
bsub -q medium -R "rusage[mem=15G]" $BASE_DIR/software/counts/run_gsea.sh \
  47681 all_unique WP_contrast 24hr_vs_Ctrl "PA" "Homo sapiens" "HALLMARK,REACTOME"

# Mouse
bsub -q medium -R "rusage[mem=15G]" $BASE_DIR/software/counts/run_gsea.sh \
  47681 all_unique WP_contrast 24hr_vs_Ctrl "" "Mus musculus" "HALLMARK,KEGG"
```

### Output

```
$BASE_DIR/<project_id>/analysis/output/counts/<bam_type>/deseq2_<contrast_table_name>/gsea_<groupB_vs_groupA>/
├── ranked_genes_<contrast_table_name>_<comparison>.rnk            # the cleaned ranking, reusable
├── gsea_<COLLECTION>_<contrast_table_name>_<comparison>.tsv       # one per collection
├── gsea_<COLLECTION>_barplot_<contrast_table_name>_<comparison>.pdf       # top 20 by NES
├── gsea_<COLLECTION>_curve_top_<contrast_table_name>_<comparison>.pdf     # enrichment curve, most significant set
├── gsea_<COLLECTION>_filtered_*.tsv / _barplot_*.pdf / _curve_top_*.pdf   # only if a filter was requested
├── gsea_summary_all_collections_<contrast_table_name>_<comparison>.tsv    # every collection, sorted by padj
└── gsea_top5_per_collection_<contrast_table_name>_<comparison>.pdf        # only when >1 collection
```

The `*.tsv` results include standard `fgsea` columns: `pathway`, `pval`,
`padj`, `NES`, `ES`, `size`, and `leadingEdge` (semicolon-separated list of
the genes driving the enrichment). The combined summary adds a `collection`
column.

The job log ends with a one-line summary of the most significant hit in each
collection, which is usually enough to decide where to look first.

> **Filenames changed.** The previous version wrote `gsea_KEGG_total_*` and
> `gsea_aminoacid_metabolism_*`. These are now `gsea_KEGG_*` and
> `gsea_<COLLECTION>_filtered_*`. Nothing downstream reads them, but older
> output folders will carry the old names.



## Changelog

### deseq2_analysis.r

All changes are backward compatible: `results_*.tsv` keeps every original
column with its original values, so `pathway_plot.r` and `gsea_analysis.r`
are unaffected.

| # | Change |
|---|---|
| 1 | Fitted object saved as `dds_<contrast_table_name>.rds`, so extra contrasts / shrinkage / re-plots no longer require re-running the pipeline. |
| 2 | Gene list parser now strips trailing inline comments (`GENE   # note`) and collapses duplicates. Annotated lists previously matched nothing and produced empty or truncated heatmaps. |
| 3 | Missing gene-list symbols are reported in two categories — absent from the annotation vs. removed by the low-count filter — which the old single warning could not distinguish. |
| 4 | New `log2FC_shrunk` / `lfcSE_shrunk` columns (`ashr`, optional). Unshrunken `log2FoldChange` and `stat` left untouched. |
| 5 | New `padj_genelist` column: BH correction over the supplied gene list only. |
| 6 | The split-variable reference level and the meaning of each interaction label are printed explicitly. Row order still determines the sign; it is now visible in the log. |
| 7 | A note is printed when any group has exactly 2 samples. |

**Known limitation, not fixed here:** counts are whole-gene including UTRs,
which is an upstream property of `get_counts_symbols.sh`. For ribosome
profiling this pools 5'UTR (uORF) reads into the gene total. CDS-restricted
counts must be produced upstream.

### gsea_analysis.r and run_gsea.sh

| # | Change |
|---|---|
| 1 | Any MSigDB collection can now be tested, not only KEGG: Hallmark, Reactome, GO (BP/CC/MF), WikiPathways, BioCarta, PID, TFT and C6, singly or several at once, via a new 7th argument. Default stays `KEGG`. |
| 2 | `run_gsea.sh` now forwards that 7th argument. The previous wrapper stopped at `species`, so a collections argument would have been silently dropped and every run would have stayed on KEGG. |
| 3 | Written against both the old and the new `msigdbr` API, detected at runtime. |
| 4 | Handles the MSigDB 2023.1 KEGG rename (`CP:KEGG` to `CP:KEGG_LEGACY`, plus the separate `CP:KEGG_MEDICUS`), reporting and choosing rather than returning an empty result. |
| 5 | Each collection is corrected for multiple testing separately, and a combined cross-collection summary table plus a top-5-per-collection plot are written. |
| 6 | The ranked list is cleaned (`NA` dropped, duplicate symbols collapsed, ties reported) and exported as `.rnk` for reuse. |
| 7 | Stops with an explicit error when `results_*.tsv` has no `stat` column, instead of failing obscurely. |
| 8 | The keyword filter now runs inside every requested collection, and gained a built-in `PA` polyamine list alongside `AA`. |
| 9 | Barplot significance colouring now uses `padj` everywhere; the filtered plot previously used raw `pval`, so the same gene set could look significant in one figure and not in the other. |
