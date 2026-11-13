# Proline-associated ribosome stalling in Ribo-seq

Codon-level Ribo-seq analysis that detects ribosome stalling at proline motifs. It can be used for eIF5A inhibition (e.g. GC7), proline limitation, or any treatment or condition expected to affect translation through prolines.

**Script:** `bj_gc7_ribosome_pausing_CDS_and_proline_genes.R`

## Quick start

```bash
module load R/4.4.3-GCCcore-14.1.0
bsub -q long -R rusage[mem=40G] Rscript $BASE_DIR/scripts/scripts_andres/bj_gc7_ribosome_pausing_CDS_and_proline_genes.R <projid> hg19 all
```

Results are written to `$BASE_DIR/<projid>/analysis/output/eIF5A_pausing/all/`. Start with `summary.txt`.

## Why

- Ribosomes form peptide bonds slowly at consecutive prolines (PP, PPP, PPG, PPD). Hypusinated eIF5A resolves these stalls and also helps termination.
- The two causes leave ribosomes in different positions, so each is read at its own position. P-sites are never shifted.
- These stalls appear in Ribo-seq as extra footprints at Pro-Pro motifs and at stop codons. Their context tells two causes apart:
  - **Less active eIF5A** (e.g. the hypusination inhibitor GC7): stalling where both the P and A sites hold Pro.
  - **Proline shortage**: stalling at any Pro codon in the A site, whatever the context.
- Typical uses:
  - Testing whether a condition (aging, senescence, stress, a drug) reduces eIF5A function.
  - Telling eIF5A defects apart from proline limitation.
  - Checking a treatment with a positive-control comparison, such as GC7 vs untreated.

**General question:** does a condition cause ribosome stalling at proline motifs, and does it resemble a known eIF5A inhibitor or proline limitation?

## Proline motifs

Motifs use the one-letter amino-acid code: **P** = proline, **G** = glycine, **D** = aspartate.

| Motif | Meaning | Why it is analysed |
|---|---|---|
| **PP** | two consecutive prolines (any context) | the general Pro-Pro stall |
| **PPP** | Pro-Pro-Pro | strongest stall; often in repetitive Pro-rich stretches with few uniquely mapped reads |
| **PPG** | Pro-Pro-Gly | among the motifs most dependent on eIF5A |
| **PPD** | Pro-Pro-Asp | among the motifs most dependent on eIF5A |

**Positions.** Positions are counted in codons from the first residue of the motif, and they always say which codon is in the **P site**; the A site is one codon further. Example for PPG:

| Position | P site | A site | Reads |
|---|---|---|---|
| −1 | residue before the motif | 1st Pro | proline limitation |
| 0 | 1st Pro | 2nd Pro | proline limitation (Pro-Pro bond) |
| **+1** | **2nd Pro** | **G** | **canonical eIF5A stall** |
| +2 | G | next residue | – |

## What it does

Two separate analyses, each with its own output folder:

| Analysis | Genes | Steps |
|---|---|---|
| **A1 — CDS** | All coding genes | **1.1** P-site QC (read lengths, 3-nt periodicity)<br>**1.2** Start/stop metaprofiles; change of initiation and termination peaks<br>**1.3** Ribosome occupancy by amino acid and codon at the E, P and A sites (comparable to diricore)<br>**1.4** uORF start vs stop peaks (uses PRICE ORFs, optional) |
| **A2 — Proline genes** | Genes classified by Pro motifs: PPP / PP only / no PP | **2.1** Pro-motif counts per gene<br>**2.2** Pausing at Pro motifs, position by position (eIF5A readout = 2nd Pro in the P site); every E-P-A tripeptide; motif classes; profiles centred on PPP, PPG, PPD and PP<br>**2.3** Gene-level footprint change by proline class<br>**2.4** Ribosome queueing before the first PPP<br>**2.5** Gene-level pause index at PP sites<br>**2.2c / 2.5** Similarity between all comparisons |

**Comparisons** are read from `rpf_density_contrasts.tsv`, the same file used by diricore (see Inputs). Each comparison is a pair of samples, test vs reference. Heatmaps 2.2c and 2.5 show how similar all comparisons are to each other: for example, replicates, time points, or a condition vs a positive control.

## How a comparison is computed

Each comparison is one test sample vs one reference sample.

1. **One transcript per gene:** the one with the most CDS P-sites.
2. **Coverage filter:** a transcript is used only if both samples have ≥ 64 P-sites in the CDS body and ≥ 0.25 P-sites per codon. The body excludes the first 15 and last 5 codons.
3. **Normalisation within each gene:** P-sites at each codon are divided by the mean P-sites per codon of that gene's CDS body. A value of 1 = an average codon of that gene, so differences in gene expression between samples cancel out.
4. **Codon by codon:** each occurrence of a motif, or each codon with a given amino acid at the E, P or A site, is a *site*. The same site is compared between test and reference.
5. **Pooling across genes:**
   - Effect = log2(mean density in test / mean density in reference) over all sites from all genes.
   - Paired Wilcoxon test over sites.
   - Effects are also shown relative to codons without Pro. Because each gene's mean is 1, extra ribosomes at some codons slightly lower all the others.

Steps 1–3 are shared by all readouts. What changes is the unit that is compared:

| Readout | Unit compared | Statistic |
|---|---|---|
| 1.2 start/stop peaks | normalised profile of each transcript, averaged | peak change; 95% bootstrap CI over transcripts |
| 1.3 occupancy by amino acid | every codon with that amino acid at E, P or A | log2 ratio of means |
| 1.4 uORF start/stop | P-sites summed over all uORFs | change of stop/start ratio; bootstrap CI over uORFs |
| 2.2 motif pausing | each motif occurrence, at each position | log2 ratio of means; paired Wilcoxon over sites |
| 2.3 gene classes | each gene (CDS P-sites per million) | log2 change per gene; Wilcoxon PPP genes vs no-PP genes |
| 2.4 queueing | each gene (density after/before the first PPP) | change per gene; Wilcoxon vs control genes |
| 2.5 gene pause index | each gene (mean density at its PP sites) | log2 ratio per gene |

## Inputs

All paths are built from `$BASE_DIR`, the project ID (used exactly as written, e.g. `17245` or `45659_AME`) and the genome:

```
$BASE_DIR/
├── static/<genome>/
│   ├── gencode.annotation.gtf
│   ├── genome.fa                                      # .fai index is created if missing
│   └── transcripts_single_header.fa
├── <projid>/analysis/
│   ├── input/periodicity_bam/<bam_type>/*.bam            # transcriptome BAMs (same as riboWaltz)
│   ├── input/metadata/rpf_density_samplenames.tsv        # sample list
│   ├── input/metadata/rpf_density_contrasts.tsv          # comparisons (same file as diricore)
│   └── output/periodicity/<bam_type>/psite_offset.tsv    # riboWaltz P-site offsets
└── <price_projid>/analysis/output/PRICE/joint_analysis/<price_projid>_<genome>_joint.orfs.tsv   # PRICE ORFs (optional, step 1.4)
```

**Sample list** (`rpf_density_samplenames.tsv`, tab-separated, no header):

```
sampleA_ID   sampleA   #cc00cc
sampleB_ID   sampleB   #668cff
```

- The first column is the BAM name without `.bam`.
- The second column is the sample name used in the comparisons.

**Comparisons** (`rpf_density_contrasts.tsv`, tab-separated, no header). One comparison per line: test, reference, and an optional colour (same format as diricore).

```
S1_treated   S0_ctrl   #cc00cc
S2_treated   S0_ctrl   #668cff
```

- Samples can be written as names or IDs from the sample list.
- If your file lists the reference first, add `--contrast_order ref_test`.

**P-site offsets.** The script uses the riboWaltz offsets in `psite_offset.tsv`. If that file is missing, it estimates them itself: for each read length, the most frequent distance from the 5′ end of the read to the start codon.

## Arguments

```
Rscript bj_gc7_ribosome_pausing_CDS_and_proline_genes.R <projid> <genome> <bam_type> [--flag value]
```

The three positional arguments are the same as in the riboWaltz script. All flags are optional.

| Flag | Default | Use |
|---|---|---|
| `--price_projid` | `<projid>` | Project where PRICE was run |
| `--price_table` | auto | PRICE table path; `none` skips step 1.4 |
| `--price_fdr` | `0.05` | FDR for PRICE ORFs (BH on `p value`) |
| `--source` | `transcriptome_bam` | `genome_bam` = genome BAMs given to PRICE, as a sensitivity check<br>`ribowaltz_rds` = saved `psite_info()` output |
| `--ribowaltz_offsets` | auto | Offsets table path; `none` = estimate offsets |
| `--min_frame0` | `0.45` | Keep read lengths with ≥ this fraction of P-sites in frame 0; `0` keeps all lengths |
| `--contrasts` | metadata file | Comparisons file |
| `--contrast_order` | `test_ref` | Column order in the comparisons file; `ref_test` if the reference comes first |
| `--sample_sheet` | – | TSV with columns `sample` and `bam`; overrides the sample list |
| `--outdir` | see above | Output folder |

## Requirements

R 4.4.3. One-time installation:

```r
install.packages(c("BiocManager", "data.table", "ggplot2", "gridExtra"))
BiocManager::install(c("GenomicAlignments", "GenomicFeatures", "txdbmaker", "rtracklayer", "Rsamtools", "Biostrings"))
```

40 GB of memory is enough. BAMs do not need to be sorted or indexed.

## Output

```
eIF5A_pausing/<bam_type>/
├── summary.txt                 # key numbers of both analyses + how to read them
├── A1_CDS/
│   ├── figures/                # 1.1 periodicity, 1.2 metaprofiles and peak changes, 1.3 occupancy, 1.4 uORF profiles
│   └── tables/
└── A2_proline_genes/
    ├── figures/                # 2.2 motif profiles, classes and similarity, 2.3 gene classes, 2.4 queueing, 2.5 similarity
    └── tables/
```

Every figure is saved as PNG and PDF, and every figure has a table with the underlying numbers.

## How to read the results

| Readout | Where | Less eIF5A activity | Proline shortage |
|---|---|---|---|
| Pausing with the 2nd Pro of PP-X in the P site (X in the A site; position +1) | A2, 2.2d | ↑ | ≈ |
| Pro occupancy at the A site (P site one codon before a Pro) | A1, 1.3 | ≈ | ↑ |
| Pausing with Pro at the A site only | A2, 2.2b | ≈ | ↑ |
| CDS termination peak | A1, 1.2 | ↑ | ≈ |
| Footprints on PPP genes vs genes without PP | A2, 2.3 | ↑ | ↑ |
| Downstream/upstream density around the first PPP | A2, 2.4 | ↓ | ↓ |

How to interpret a comparison:

1. **Positive control.** A positive-control comparison (e.g. GC7 vs untreated) should show these changes, in every replicate if there are replicates. This shows the assay works.
2. **Condition of interest.** A condition with the same pattern behaves like reduced eIF5A activity. A pattern dominated by Pro in the A site regardless of context points to proline limitation.
3. **uORFs (1.4).**
   - A stop/start ratio above 0 means ribosomes leave uORFs more slowly (termination).
   - A ratio below 0 means relatively more initiation at uORFs.

## Notes

- **Replicates are optional.** Each comparison is one test sample vs one reference sample, so the analysis also runs without replicates.
  - Log2 changes and profiles give the direction and size of an effect.
  - P-values and confidence intervals measure how consistent the effect is across sites or transcripts within that comparison. They do not capture biological variability.
  - Without replicates, read the results as exploratory. With replicates, check that the effects agree; the similarity heatmaps (2.2c, 2.5) help with this.
  - Comparisons that share a reference sample are partly correlated in these heatmaps because of that shared sample.
- **PPP motifs** often lie in repetitive Pro-rich stretches with few uniquely mapped reads; rely mainly on PPG, PPD and PP.
- **Low-depth samples.** If a shallow sample leaves too few transcripts after filtering, lower `MIN_BODY_DENSITY` at the top of the script.
- **Transcriptome BAMs.** Reads mapping to several isoforms are kept, as in riboWaltz. Each gene is analysed on a single transcript, the one with the most CDS P-sites, so no read is counted twice within a gene.
- **Validation.** The script was tested on simulated data with known pauses. It recovered the planted effects, and transcriptome and genome BAM inputs gave identical results.
