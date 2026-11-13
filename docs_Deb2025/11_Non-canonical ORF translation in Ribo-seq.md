# Non-canonical ORF translation in Ribo-seq

Measures how much non-canonical ORFs (uORFs, uoORFs, iORFs, dORFs, ORFs in ncRNAs) are translated in each comparison, using PRICE ORFs and Ribo-seq gene counts. It also reports marker genes (eIF5A/polyamine axis, stress response, proliferation) and pathway changes. Replicates are not required.

**Script:** `noncanonical_orfs_riboseq.R`

## Run

```bash
module load R/4.4.3-GCCcore-14.1.0
bsub -q long -R rusage[mem=40G] Rscript $BASE_DIR/scripts/scripts_andres/noncanonical_orfs_riboseq.R <projid> <genome> <count_type> <deseq2_run>
# e.g. ... 17245_48138 hg19 all_unique AB_contrast
```

- `<count_type>` and `<deseq2_run>` are the values given to `run_deseq2.sh`.
- Optional flags:
  - `--price_projid` if PRICE was run under another project.
  - `--contrast_order ref_test` if the comparisons file lists the reference first.
  - `--counts`, `--price_table` and `--outdir` to give paths explicitly.

## Inputs

```
$BASE_DIR/
├── static/<genome>/gencode.annotation.gtf, genome.fa
├── <projid>/analysis/input/metadata/
│   ├── rpf_density_contrasts.tsv      # test <tab> reference [<tab> colour]  (same as diricore)
│   └── rpf_density_samplenames.tsv    # ID <tab> name <tab> colour (optional)
├── <projid>/analysis/output/counts/<count_type>/deseq2_<deseq2_run>/raw_counts_matrix_<deseq2_run>.tsv
└── <price_projid>/analysis/output/PRICE/joint_analysis/   # processed *_joint*deseq2.tsv, or <price_projid>_<genome>_joint.orfs.tsv
```

The PRICE table needs one activity column per sample.

## What it does

For each comparison (one test sample vs one reference sample):

1. **Genes:** log2FC after removing mitochondrial genes, and GSEA (Hallmarks, Reactome) on genes ranked by log2FC.
2. **Non-canonical ORFs:** PRICE ORFs without CDS, Trunc or mitochondrial ORFs, with two normalisations:
   - **Depth:** ORF activity divided by the gene-level size factors.
   - **Host gene:** each ORF relative to its own gene (the stricter measure).
3. **Readouts:**
   - Sample-level non-canonical translation.
   - Change per ORF type.
   - ORFs detected at equal sequencing depth.
   - Controls: AUG vs non-AUG uORFs, ORF and gene length.
4. **Across comparisons:** PCA, sample correlations, and similarity between comparisons (e.g. replicates or time points).

## Output

`$BASE_DIR/<projid>/analysis/output/noncanonical_orfs/<count_type>/`

```
report_noncanonical_orfs.pdf   # summary + 11 figures with captions
summary.txt                    # key numbers per comparison
materials_and_methods.md       # methods of this run: inputs, comparisons, numbers, versions
tables/                        # per comparison: genes, ORFs, ORF types, GSEA; per-sample levels
figures/                       # PNG + PDF
```

## Notes

- Each comparison is one sample vs one sample, so p-values measure consistency across ORFs within that comparison, not biological variability.
- Without replicates, read the results as exploratory. With replicates, check that they agree (figure 09).
- For a replicated 2 × 2 design with DESeq2 and a batch term, see `bj_senescence_gc7_noncanonical_orfs.R`.
