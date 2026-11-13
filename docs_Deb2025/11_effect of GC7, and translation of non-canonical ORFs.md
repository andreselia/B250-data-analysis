# Materials and Methods

**BJ fibroblasts: proliferating vs senescent, effect of GC7, and translation of non-canonical ORFs (Ribo-seq + PRICE)**

Project 17245_48138 · genome hg19 · analysis version 2 (mitochondrial genes excluded; non-canonical ORFs normalised by gene-level depth and by host gene)

All results, tables and figures described here are produced by a single R script, `bj_senescence_gc7_noncanonical_orfs.R`. The PDF report only arranges those figures and quotes the numbers written by the script to `metrics_summary.json`.

---

## 1. Experimental design and input data

### 1.1 Samples

Human BJ fibroblasts were analysed in two states, proliferating (**pro**) and senescent (**sen**), each either untreated (**ctrl**) or treated with the deoxyhypusine synthase inhibitor GC7 (**gc7**). There are two biological replicates per condition. The replicates were prepared with the same library protocol but sequenced in different years, so replicate and sequencing batch are the same factor.

| Sample | State | Treatment | Replicate (batch) |
|---|---|---|---|
| BJ_pro | pro | ctrl | rep2 |
| BJ_pro_gc7 | pro | gc7 | rep2 |
| BJ_sen | sen | ctrl | rep2 |
| BJ_sen_gc7 | sen | gc7 | rep2 |
| BJ_pro1 | pro | ctrl | rep1 |
| BJ_pro1_gc7 | pro | gc7 | rep1 |
| BJ_sen1 | sen | ctrl | rep1 |
| BJ_sen1_gc7 | sen | gc7 | rep1 |

Each replicate contains all four conditions, so batch is not confounded with state or treatment.

### 1.2 Upstream processing

The following steps were performed before this analysis. The bracketed fields should be completed with the exact parameters used.

- Ribo-seq library preparation: [kit / protocol].
- Adapter trimming and size selection of footprints: [tool, adapter sequence, length range].
- Removal of rRNA/tRNA reads: [tool, reference].
- Alignment to hg19: [aligner, version, parameters]; gene annotation: [annotation and version].
- PCR-duplicate removal using unique molecular identifiers (UMIs): [tool, e.g. UMI-tools, method].
- Gene-level counts of uniquely mapped reads ("all_unique"): [counting tool and parameters].
- ORF identification and quantification with PRICE (Erhard et al., 2018), run jointly on all eight samples: [GEDI/PRICE version, parameters].

### 1.3 Input files

| File | Content |
|---|---|
| `raw_counts_matrix_AB_contrast.tsv` | Raw (UMI-deduplicated) footprint counts per gene and sample. |
| `17245_48138_hg19_joint_gc7_vs_ctrl_deseq2.tsv` | PRICE joint output. Columns used: `Id`, `gene_symbol`, `Type`, `Codon` (start codon), `Location`, `price_padj`, `<sample>` (ORF activity), `<sample>_norm` (activity divided by the gene-level size factor), `<sample>_gene_norm` (normalised host-gene counts) and `<sample>_ratio` (= `_norm / _gene_norm`). |
| `gene_length_gc.tsv` | Exonic length (union of exons) and GC content per gene symbol. The cluster version computes it once from `$BASE_DIR/static/<genome>/gencode.annotation.gtf` and `genome.fa` and saves it in `tables/`. The original analysis used GENCODE v22 values from TCGAbiolinks `geneInfoHT`. Used only for the length control (Figure 20). |

---

## 2. Software

All statistics and figures were produced in R 4.4.3 with DESeq2 1.46.0, limma 3.62.1, fgsea 1.32.2, msigdbr 7.5.1 (MSigDB v7.5.1 gene sets, bundled offline), ashr 2.2-63, ggplot2 4.0.3, patchwork 1.3.2, pheatmap 1.0.13, ggrepel 0.9.8, hexbin 1.28.6, gridExtra 2.3.1 and jsonlite 2.0.0. The complete list of package versions is written to `sessionInfo.txt`. The PDF report was assembled in Python with reportlab (layout only; no calculations).

Random seeds: 42 for GSEA, 1 for rarefaction.

---

## 3. Gene-level analysis

### 3.1 Exclusion of mitochondrial genes

Because it is not known whether sample collection captured mitochondrially encoded transcripts consistently, mitochondrial features were removed **before any normalisation**. A gene was removed if its symbol matched the regular expression `(^|_)(MT-|MT(ND|CO|ATP|CY|RNR|T[A-Z])[0-9A-Z]*P[0-9]+|MTRNR2L)`. This removes 183 genes: the 37 mtDNA-encoded genes (MT-), nuclear mitochondrial pseudogenes (NUMTs, e.g. MTND1P23, MTCO1P12), whose reads are mostly mis-assigned mitochondrial reads, and the MTRNR2L family.

### 3.2 Filtering and differential analysis

Genes with at least 10 reads in at least 2 samples were kept (21,536 genes). Differential translation was tested with DESeq2 using the design `~ batch + cond`, where `batch` is the replicate (reference rep1) and `cond` combines state and treatment (`pro_ctrl` reference, `pro_gc7`, `sen_ctrl`, `sen_gc7`). Size factors were estimated with the median-of-ratios method. Four contrasts were tested with Wald tests on linear combinations of coefficients:

| Contrast | Definition | Meaning |
|---|---|---|
| `sen_vs_pro` | sen_ctrl − pro_ctrl | effect of senescence in untreated cells |
| `gc7_in_pro` | pro_gc7 − pro_ctrl | effect of GC7 in proliferating cells |
| `gc7_in_sen` | sen_gc7 − sen_ctrl | effect of GC7 in senescent cells |
| `interaction` | (sen_gc7 − sen_ctrl) − (pro_gc7 − pro_ctrl) | difference between the GC7 effects in the two states |

P-values were adjusted with the Benjamini–Hochberg procedure; padj < 0.05 was considered significant. Log2 fold changes shrunken with ashr (Stephens, 2017) are reported alongside the raw estimates and are used only for display (volcano plot).

### 3.3 Transformation for visualisation

Counts were transformed with the variance-stabilising transformation (`vst`, `blind = FALSE`). For plots only, the batch effect was removed from the VST matrix with `limma::removeBatchEffect`, passing `design = ~ cond` so that condition effects are preserved. Batch-removed values were never used for testing; tests always include batch in the DESeq2 model.

### 3.4 Gene set enrichment analysis

Genes were ranked by the DESeq2 Wald statistic of each contrast and analysed with fgsea (`minSize = 10`, `maxSize = 500`, `eps = 0`) against three collections, each corrected for multiple testing separately: the 50 MSigDB Hallmark sets, the Reactome sets of MSigDB C2:CP, and a focused list of senescence, polyamine, translation and stress sets (FRIDMAN_SENESCENCE_UP/DN, REACTOME_CELLULAR_SENESCENCE, REACTOME_SENESCENCE_ASSOCIATED_SECRETORY_PHENOTYPE_SASP, REACTOME_METABOLISM_OF_POLYAMINES, REACTOME_EUKARYOTIC_TRANSLATION_INITIATION, REACTOME_RESPONSE_OF_EIF2AK4_GCN2_TO_AMINO_ACID_DEFICIENCY, HALLMARK_E2F_TARGETS, HALLMARK_G2M_CHECKPOINT, HALLMARK_INFLAMMATORY_RESPONSE, HALLMARK_UNFOLDED_PROTEIN_RESPONSE, HALLMARK_P53_PATHWAY, HALLMARK_MTORC1_SIGNALING, IGARASHI_ATF4_TARGETS_DN).

### 3.5 Marker panels and per-sample scores

Senescence markers were grouped as proliferation/cell cycle, cell-cycle arrest (p53/p21/p16) and SASP/secretome. Per-sample scores for E2F targets, G2M checkpoint and FRIDMAN_SENESCENCE_UP are the mean of the gene-wise z-scores of the batch-removed VST.

The eIF5A axis comprises EIF5A, EIF5AL1 and EIF5A2; the hypusination enzymes DHPS and DOHH; polyamine synthesis and its regulation (ODC1, AZIN1, ADC/AZIN2, OAZ1, OAZ2, AMD1, SRM, SMS, MTAP); polyamine catabolism and transport (SAT1, SAT2, SMOX, PAOX, ATP13A2, ATP13A3, SLC3A2); and integrated-stress-response (ATF4) targets (ATF4, DDIT3, ASNS, TRIB3, CHAC1, SESN2, PPP1R15A). The ISR score is the mean of the row-centred, uncorrected VST of ATF4, DDIT3, ASNS, TRIB3, CHAC1 and SESN2. Batch was not removed for GC7-response markers because the response differs between replicates.

Note that Ribo-seq measures translation (ribosome footprints), not hypusination. The level of hypusinated eIF5A cannot be inferred from these data.

---

## 4. ORF-level analysis

### 4.1 ORF selection

ORFs with PRICE `price_padj < 0.05` were kept. CDS and Trunc (N-terminally truncated CDS) ORFs were removed, as were all mitochondrial ORFs (chromosome MT or host symbol matching the pattern in 3.1; 19 ORFs). This leaves 4,386 ORFs: uORF 1,687, ncRNA 1,240, Variant 737, uoORF 342, iORF 265, dORF 98 and orphan 17.

- **Non-canonical ORFs**: uORF (upstream ORF in the 5′ UTR), uoORF (upstream ORF overlapping the annotated CDS), iORF (internal ORF in a different frame within the CDS), dORF (downstream ORF in the 3′ UTR), ncRNA (ORF in an annotated non-coding RNA) and orphan (ORF without an annotated host).
- **Variant** ORFs (isoforms of annotated CDSs with different starts or splicing) are dominated by highly translated genes (e.g. THBS1, SPARC, FN1). They are reported separately and are not counted as non-canonical.

PRICE ORF activities were rounded to integers for DESeq2. ORF length was computed from `Location` as the sum of its exon blocks.

### 4.2 Normalisation A — gene-level sequencing depth

The gene-level DESeq2 size factors (Section 3.2, without mitochondrial genes) were assigned, fixed, to the ORF count matrix. They differ by at most 2.5% from the size factors implied by the `_norm` columns of the PRICE table (`activity / _norm`), which were computed by the upstream pipeline on all genes. This normalisation corrects only for sequencing depth, so a change in an ORF is expressed relative to the typical gene. ORFs with ≥10 reads in ≥2 samples were tested (3,132 ORFs) with the same design and contrasts as the genes. A VST of these counts was used for PCA, correlations and heatmaps.

### 4.3 Normalisation B — host gene

Host-gene raw counts were reconstructed as `_gene_norm × size factor of the PRICE table`. For each ORF, the DESeq2 normalisation factors were set to its host-gene counts divided by their geometric mean across samples. With this offset, the fitted log2 fold change is the change in the **ORF / host-gene ratio**, the model-based equivalent of the `_ratio` columns. ORFs passing the filter of 4.2 whose host gene had ≥10 reads in every sample were tested (2,620 ORFs). About 420 ncRNA ORFs have no host gene in the count matrix and are absent from this normalisation only.

### 4.4 Normalisations deliberately not used

- **PRICE CDS calls**: these are predictions of PRICE, not independent measurements of canonical translation.
- **Total of non-canonical ORFs**: dividing by this total assumes that the treatments do not change the global level of non-canonical translation, which is the quantity being tested; it would force the median change to zero.

---

## 5. Summaries of non-canonical ORFs

### 5.1 Per-sample level, gene-level depth (Figure 8)

For each sample and ORF type, the sum of PRICE activities of all non-canonical ORFs of that type divided by the gene-level size factor (in thousands). This equals the sum of the `_norm` columns.

### 5.2 Per-sample level relative to host gene (Figure 9)

For ORFs whose host gene has ≥10 reads in all samples and whose total activity is ≥40 (2,007 ORFs), log2 ratios were computed as log2(ORF + 0.5) − log2(host + 0.5). The per-sample value is the median ratio across ORFs, overall and by type.

### 5.3 Sample-level model

Each per-sample metric was modelled with `lm(y ~ batch + cond)` on the eight samples (3 residual degrees of freedom). For the depth metric, y = log2(level); for the host metric, y is already a log2 ratio. The three biological contrasts of Section 3.2 were estimated as linear combinations of coefficients and tested with two-sided t-tests. Estimates are reported as percent change, 100 × (2^estimate − 1).

### 5.4 log2FC distributions by ORF type (Figure 10)

For ORFs with DESeq2 baseMean ≥ 10, the distribution of log2FC was summarised by type. With the depth normalisation, each type was compared with the log2FC distribution of all genes (baseMean ≥ 10) using a two-sided Mann-Whitney test. With the host-gene normalisation, each type was compared with 0 using a two-sided Wilcoxon signed-rank test. These tests treat ORFs as independent observations within the same samples; they measure how consistently ORFs shift, not reproducibility across biological replicates.

### 5.5 Per-replicate consistency (Figure 11)

Each replicate was analysed separately for each comparison (e.g. BJ_pro1_gc7 vs BJ_pro1 in rep1). Depth: for ORFs with ≥20 reads summed over the two samples, the median difference of log2(activity / size factor + 1) per type minus the median difference of log2(normalised count + 1) of genes with mean normalised count > 20. Host gene: the median difference of the log2 ORF/host ratios (Section 5.2) for ORFs with ≥20 reads in the pair.

### 5.6 Detection after rarefaction (Figure 12)

To compare the number of detectable non-canonical ORFs at equal sequencing depth, each sample's rounded ORF activities were binomially thinned with probability (smallest gene-level size factor / size factor of the sample), i.e. to the depth of BJ_pro. An ORF was called detected with ≥3 footprints. The procedure was repeated 20 times and averaged.

### 5.7 PCA, correlations and heatmaps (Figures 1, 2, 13–16)

PCA used the 500 most variable rows (centred, not scaled; `prcomp`). For ORF/host ratios, the batch was removed by subtracting the per-replicate row mean. Sample correlations are Spearman correlations of batch-removed VST values (genes: 1,000 most variable; non-canonical ORFs: all). Heatmaps show row z-scores of batch-removed VST values.

### 5.8 Controls for the GC7 effect in proliferating cells (Figure 20)

- **Length**: median depth-normalised log2FC (GC7 in pro) by length bin, for non-canonical ORF types (ORF length) and for genes (exonic length from `gene_length_gc.tsv`, genes with baseMean ≥ 50). Spearman correlations of log2FC with length were computed for uORFs and for genes.
- **Start codon**: log2FC of uORFs and uoORFs with AUG vs non-AUG start codons, compared with a Mann-Whitney test.

### 5.9 Does GC7 resemble senescence? (Figure 21)

Correlating "GC7 in pro" with "sen vs pro" within the same replicate shares the proliferating control sample in both differences, which induces a spurious positive correlation. The comparison was therefore made across replicates (GC7 in pro from rep1 vs senescence from rep2, and vice versa) using VST differences and Spearman correlations, for genes (mean normalised count > 50) and for non-canonical ORFs. The per-ORF reproducibility of the GC7 effect was assessed by correlating the rep1 and rep2 GC7-in-pro differences.

---

## 6. Limitations

- Two biological replicates per condition. Sample-level tests have 3 residual degrees of freedom, and no individual ORF reaches significance.
- The response to GC7 differs between replicates: the integrated stress response is induced only in rep1.
- BJ_pro has much lower depth (1.8 M reads) than the other samples and is the proliferating control of rep2.
- Hypusination was not measured; Ribo-seq gene-level data reflect translation, not protein modification.
- The host-gene normalisation excludes ncRNA ORFs without a host gene in the count matrix.
- Gene lengths come from GENCODE v22 (hg38 annotation, symbol-mapped) and are used only as an approximate control.

---

## 7. Figure guide

**How to read the per-sample plots (Figures 6, 8, 9, 12, 18).** Each point is one sample. Blue = proliferating, orange = senescent; filled = rep1 (names ending in "1"), open = rep2; circle = control, triangle = GC7. Coloured lines join control and GC7 samples of the same state and replicate (solid = rep1, dashed = rep2); grey lines join proliferating and senescent controls of the same replicate. A treatment effect that replicates appears as two coloured lines with the same slope. Where present, the text under a panel gives the sample-level model estimates (Section 5.3).

| Figure | What it shows | How to read it / main result |
|---|---|---|
| 1. PCA of genes | PCA of the 500 most variable genes, uncorrected and batch-removed. | Uncorrected, PC1 separates replicates (batch is the largest source of variation). Batch-removed, PC1 separates proliferating from senescent cells. |
| 2. Gene correlation heatmap | Spearman correlations between samples (1,000 most variable genes, batch-removed), hierarchically clustered. | Samples cluster by state, confirming that senescence dominates once batch is removed. |
| 3. Senescence markers | Row z-scores of marker genes; the number after each gene is log2FC sen vs pro (controls), * padj < 0.05. | Proliferation genes are lower and CDKN1A, MDM2, MMP1, MMP3 higher in senescent cells. CDKN2A differs mainly between batches. |
| 4. Volcano, sen vs pro | Shrunken log2FC vs −log10 p for untreated senescent vs proliferating cells. | 1,012 genes with padj < 0.05 (590 up, 422 down). |
| 5. GSEA, sen vs pro | Top significant Hallmark and Reactome sets (padj < 0.05); bar = NES, label = padj. | Cell-cycle sets fall; p53, inflammation and secretion rise: a typical senescence profile. |
| 6. Senescence scores | Per-sample mean z for E2F targets, G2M checkpoint and FRIDMAN_SENESCENCE_UP. | Proliferation scores fall in senescence in both replicates and fall further with GC7 in senescent cells. |
| 7. ORF types | Number of PRICE-significant ORFs by type, and per-sample depth-normalised non-canonical level by type. | uORFs and ncRNA ORFs dominate the non-canonical repertoire. |
| 8. Non-canonical level, depth | Per-sample level by type (Section 5.1). | GC7 raises uORF and uoORF levels in proliferating cells in both replicates; senescence does not meaningfully change them. |
| 9. Non-canonical level, host gene | Per-sample median log2(ORF/host) by type (Section 5.2). | The most stringent measure: GC7 in pro +25% overall (p = 0.009); senescence ~0. |
| 10. log2FC by ORF type | Boxplots of per-ORF log2FC for each contrast; top = depth normalisation vs all genes; bottom = host-gene normalisation vs 0. Labels = median and significance. | uORFs, uoORFs and iORFs shift upwards only with GC7 in proliferating cells; the interaction row shows that this shift is absent in senescent cells. |
| 11. Per-replicate consistency | Heatmaps of median per-type changes for each replicate and comparison, with both normalisations. | The GC7 increase in proliferating cells is present in rep1 and rep2; senescence relative to host genes stays near 0. |
| 12. Detection after rarefaction | Number of non-canonical ORFs and uORFs detected at equal gene-level depth. | GC7 increases detection in proliferating cells in both replicates; senescence only slightly (+3–8%). |
| 13. PCA of non-canonical ORFs | PCA of depth-normalised VST of non-canonical ORFs, uncorrected and batch-removed. | PC1 separates states; PC2 moves both GC7-treated proliferating samples in the same direction. |
| 14. PCA of uORFs and host ratios | Left: uORFs + uoORFs only. Right: non-canonical ORFs as log2 ORF/host ratios. | Once host-gene expression is removed, the separation by state weakens: state differences in ORFs largely follow their host genes. |
| 15. Non-canonical ORF correlation heatmap | Spearman correlations between samples using non-canonical ORFs. | Complements Figure 13. |
| 16. Top non-canonical ORFs | Row z-scores of the 60 most variable non-canonical ORFs, annotated by type and start codon. | Shows which individual ORFs drive the sample structure; individual changes are not reproducible between replicates (Figure 21). |
| 17. eIF5A–polyamine axis | Left: z-scores per sample. Right: log2FC per contrast (colour), −log10 p (size), * padj < 0.05. | eIF5A, DHPS and DOHH do not change significantly; SAT1 and SMOX trend up in senescence; ISR genes rise mainly in BJ_pro1_gc7. |
| 18. GC7-response markers | ISR score, AMD1 and SAT1 per sample (uncorrected). | ISR induction is restricted to rep1; AMD1 decreases with GC7 in proliferating cells in both replicates. |
| 19. GSEA, GC7 | Top Hallmark sets for GC7 in pro and in sen. | In senescent cells, GC7 lowers E2F, G2M and MYC targets. |
| 20. Length and start-codon controls | Left: median log2FC (GC7 in pro) by length bin for ORF types and genes. Right: AUG vs non-AUG uORFs/uoORFs. | uORF/uoORF/iORF increases are present at all lengths, and short genes do not gain footprints; AUG and non-AUG uORFs increase similarly (p = 0.25). |
| 21. Effect correlations | Hexbin plots of GC7-in-pro vs sen-vs-pro differences across replicates (genes, non-canonical ORFs), and of GC7-in-pro in rep1 vs rep2 (red dot = medians). | ρ ≈ 0: GC7 does not mimic senescence. The per-ORF GC7 effect is not reproducible (ρ ≈ 0.01), but the global shift is (both medians > 0). |

---

## 8. Output files

All tables are tab-separated and written to `results/tables/`.

| File | Content |
|---|---|
| `samples.tsv` | Sample sheet (batch, state, group, cond). |
| `size_factors.tsv` | Gene-level size factors without mitochondrial genes vs those implied by the PRICE `_norm` columns. |
| `genes_DE_<contrast>.tsv` | Gene-level DESeq2 results: `baseMean`, `log2FC`, `log2FC_shrunk` (ashr), `lfcSE`, `stat`, `pvalue`, `padj`. |
| `genes_vst.tsv`, `genes_vst_batch_removed.tsv`, `genes_normalized_counts.tsv` | Transformed and normalised gene matrices. |
| `orfs_DE_depth_<contrast>.tsv` | ORF-level DESeq2 with gene-level depth normalisation, with ORF annotation (`Id`, `gene_symbol`, `Type`, `class`, `Codon`, `Location`, `len_nt`). |
| `orfs_DE_hostgene_<contrast>.tsv` | ORF-level DESeq2 with host-gene normalisation; `log2FC` = change in ORF/host ratio. |
| `orfs_vst_depth.tsv`, `orfs_vst_depth_batch_removed.tsv` | VST of ORF counts (depth normalisation). |
| `orfs_hostgene_counts.tsv` | Reconstructed raw host-gene counts per ORF. |
| `gsea_<collection>_<contrast>.tsv` | fgsea results (`hallmark`, `reactome`, `focus`), with the first 25 leading-edge genes. |
| `noncanonical_level_per_sample_depth.tsv`, `noncanonical_level_per_sample_hostgene.tsv` | Per-sample metrics of Sections 5.1 and 5.2. |
| `consistency_depth_per_replicate.tsv`, `consistency_hostgene_per_replicate.tsv` | Per-replicate medians of Section 5.5. |
| `detection_rarefied.tsv` | Mean number of detected ORFs per type and sample after rarefaction. |

`<contrast>` is one of `sen_vs_pro`, `gc7_in_pro`, `gc7_in_sen`, `interaction`. Figures are written to `results/figures/` as PNG (200 dpi) and PDF. `metrics_summary.json` contains every number quoted in the report, and `sessionInfo.txt` the package versions.

---

## 9. Reproducing the analysis

On the cluster (same conventions as the pausing script):

```
module load R/4.4.3-GCCcore-14.1.0
bsub -q long -R rusage[mem=40G] Rscript $BASE_DIR/scripts/scripts_andres/bj_senescence_gc7_noncanonical_orfs.R <projid> hg19
```

- **Inputs found automatically:** the gene count matrix (`raw_counts_matrix*.tsv` under `$BASE_DIR/<projid>/analysis/output/`) and the processed PRICE table (`*_joint*deseq2.tsv` in `$BASE_DIR/<price_projid>/analysis/output/PRICE/joint_analysis/`).
- **Optional flags:** `--price_projid`, `--counts`, `--price_table`, `--gene_length`, `--outdir`.
- **Output:** written to `$BASE_DIR/<projid>/analysis/output/noncanonical_orfs/<genome>/`, including `report_noncanonical_orfs.pdf` (key numbers and the 21 figures with captions).
- **Run time:** a few minutes.
- **Expected small differences:** rarefaction counts (random thinning) and GSEA NES values across fgsea versions.

---

## References

Please verify these references before use; they were compiled without access to a bibliographic database.

- Erhard F, Halenius A, Zimmermann C, et al. Improved Ribo-seq enables identification of cryptic translation events. *Nat Methods* 2018;15:363–366. (PRICE)
- Love MI, Huber W, Anders S. Moderated estimation of fold change and dispersion for RNA-seq data with DESeq2. *Genome Biol* 2014;15:550.
- Stephens M. False discovery rates: a new deal. *Biostatistics* 2017;18:275–294. (ashr)
- Ritchie ME, Phipson B, Wu D, et al. limma powers differential expression analyses for RNA-sequencing and microarray studies. *Nucleic Acids Res* 2015;43:e47.
- Korotkevich G, Sukhov V, Budin N, et al. Fast gene set enrichment analysis. *bioRxiv* 2016/2021, doi:10.1101/060012. (fgsea)
- Subramanian A, Tamayo P, Mootha VK, et al. Gene set enrichment analysis: a knowledge-based approach for interpreting genome-wide expression profiles. *Proc Natl Acad Sci USA* 2005;102:15545–15550.
- Liberzon A, Birger C, Thorvaldsdóttir H, et al. The Molecular Signatures Database Hallmark gene set collection. *Cell Syst* 2015;1:417–425.
- Fridman AL, Tainsky MA. Critical pathways in cellular senescence and immortalization revealed by gene expression profiling. *Oncogene* 2008;27:5975–5987.
- Colaprico A, Silva TC, Olsen C, et al. TCGAbiolinks: an R/Bioconductor package for integrative analysis of TCGA data. *Nucleic Acids Res* 2016;44:e71. (gene length and GC table)
