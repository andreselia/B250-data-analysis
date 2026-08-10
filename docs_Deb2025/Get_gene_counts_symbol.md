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
 
1. Reads a contrast table (see format below) and restricts the analysis to
   **only** the samples listed in it -- any other sample present in the
   counts folder (e.g. from a different batch) is ignored.
2. Builds a raw counts matrix from the per-sample `*.gene_counts.tsv` files.
3. Runs `DESeq2` (`~group` design), after a light low-count filter
   (`rowSums >= 10`).
4. Exports **every pairwise comparison** between the groups present in the
   contrast table (not just one contrast), e.g. for groups
   `Ctrl`, `2hr`, `24hr` it produces `2hr_vs_Ctrl`, `24hr_vs_Ctrl`, and
   `24hr_vs_2hr`.
5. Exports normalized/variance-stabilized (VST) counts for all genes, and a
   plain size-factor-normalized (linear-scale) counts table.
6. Given a user-supplied gene list, subsets the VST-normalized matrix,
   exports it separately, and produces a faceted boxplot (one panel per
   gene, samples grouped by `group`) covering every sample in the contrast
   table.
### Batches / multiple contrast tables
 
Contrast tables live at a fixed, predictable path:
 
```
$BASE_DIR/<project_id>/analysis/input/metadata/<contrast_table_name>.tsv
```
 
Because the sample selection is entirely driven by the contrast table, you
can keep multiple independent analyses (e.g. two sequencing batches, or two
different sample groupings) side by side in the same project without any
batch-effect cross-contamination: each run only ever sees the samples in the
table it was given. All output files and their parent folder are suffixed
with `<contrast_table_name>`, so results from different batches/contrast
tables never overwrite each other.
 
### Contrast table format
 
Tab-separated, with header, columns `sampleid` and `group`:
 
```
sampleid	group
HEK293_Ctrl_1	Ctrl
HEK293_2hr_1	2hr
HEK293_24hr_1	24hr
HEK293_Ctrl_2	Ctrl
HEK293_2hr_2 	2hr
HEK293_24hr_2	24hr

```
 
`sampleid` must match the prefix used by `get_counts_symbols.sh`
(i.e. `<sampleid>.gene_counts.tsv` must exist in the counts folder).
 
### Gene list format
 
Plain text, one gene symbol per line (blank lines and lines starting with
`#` are ignored):
 
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
| `project_id` | Project folder name under `$BASE_DIR` | required |
| `bam_type` | Must match the `bam_type` used in `get_counts_symbols.sh` (`all`, `all_unique`, `hq_unique`) | required |
| `contrast_table_name` | Name (no path, no `.tsv`) of the contrast table under `analysis/input/metadata/` | `deseq2_contrast_table` |
| `gene_list_path` | Full path to the gene list file | required |
| `genome` | Only used for logging | `hg19` |
 
Example -- two batches in the same project:
 
```bash
# Batch 1
bsub -q medium -R "rusage[mem=30G]" $BASE_DIR/software/counts/run_deseq2.sh \
  47681 all_unique deseq2_contrast_table_batch1 \
  $BASE_DIR/47681/analysis/input/metadata/gene_list.txt
 
# Batch 2
bsub -q medium -R "rusage[mem=30G]" $BASE_DIR/software/counts/run_deseq2.sh \
  47681 all_unique deseq2_contrast_table_batch2 \
  $BASE_DIR/47681/analysis/input/metadata/gene_list.txt
```
 
### Output
 
```
$BASE_DIR/<project_id>/analysis/output/counts/<bam_type>/deseq2_<contrast_table_name>/
├── raw_counts_matrix_<contrast_table_name>.tsv
├── results_<contrast_table_name>_<groupB>_vs_<groupA>.tsv     # one per pair of groups
├── normalized_transformed_counts_<contrast_table_name>.tsv    # VST, all genes
├── normalized_counts_linear_<contrast_table_name>.tsv         # size-factor normalized, linear scale
├── selected_genes_normalized_transformed_counts_<contrast_table_name>.tsv
└── selected_genes_plot_<contrast_table_name>.pdf
```
 
All output paths are printed to stdout (visible in the `bsub` job log) at
the end of the run.
 
### Requirements
 
- R with `DESeq2` and `ggplot2` installed (loaded here via
  `module load R/4.4.3-GCCcore-14.1.0`, adjust if your environment differs).
---

