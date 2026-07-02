# Diricore on subset of genes

Let's consider the following list of Mitochondrial genes: 

```
ERG
MST1R
TPO
TMC5
ZIC1
KIAA1683
CACNA1B
ABCG4
LINGO2
IFI27
ELN
DCDC2
DMKN
GNA15
SYTL4
POU2F2
....
```


## 0. Create a reference file with your gene list

```
vim $BASE_DIR/Blanco26/analysis/output/TE-DOWNtranscripts.txt
```

## 1. Load samtools

```
module load SAMtools/1.20-GCC-14.1.0 

```

## 2. Extract reads from bam: 

This is an updated script. It takes your list, obtains the coordinates for each gene and generates a bed file in the tmp folder.
Once the bed file is generated, it produces one script per sample to directly extract the reads from genome-aligned reads

```
$BASE_DIR/software/diricore_subset/1_extract_bam_v4.sh Blanco26 all $BASE_DIR/Blanco26/analysis/output/TE-DOWNtranscripts.txt
```

Total reads in BAM: 15204
Stats saved: /omics/groups/OE0532/internal/Andres//Blanco26/analysis/output/diricore_subset/all_TE-DOWNtranscripts/alignments/toGenome/LN229_WT2_30_readstats.tsv

## 3. Run rpf density analysis: 

```
bsub -q long -R "rusage[mem=20G]" $BASE_DIR/software/diricore_subset/rpf_density_analysis.sh 18436 hg19 5 all_MT-transcripts
```

## 4. Subsequence analysis:

```
bsub -q long -R "rusage[mem=20G]" $BASE_DIR/software/diricore_subset/subsequence_analysis.sh 18436 hg19 5 all_MT-transcripts
```

## 5. RPF transcript distribution:

```
bsub -q long  -R "rusage[mem=20G]" $BASE_DIR/software/diricore_subset/plot_rpf_transcript_distribution.sh 18436 hg19 5 all_MT-transcripts
```

# Diricore EXCLUDING subset of genes

Let's say we want to exclude mitochondial genes from the analysis.

## 1. Extract reads from bam: 

```
$BASE_DIR/software/diricore_subset/1_extract_bam.sh 22276 all $BASE_DIR/static/hg19/MT-transcripts.txt exclude
```

New bam files will be written to: `$BASE_DIR/22276/analysis/output/diricore_subset/all_excl_MT-transcripts/alignments/toGenome`

Perform the analysis as described above.

## 2. Difference between MT and cytosolic genes in subsequence analysis

Because a few codons are different in cytosolic genes and MT-genes, there are 2 subsequence scripts. They output the same data files, but the plotting function is different due to the codons annotation. 

### 2.1 Subsequence plots for the cytosolic genes: 

```
bsub -q long -R "rusage[mem=20G]" $BASE_DIR/software/diricore_subset/subsequence_analysis_MT.sh 18436 hg19 5 all_MT-transcripts
```

### 2.2 Subsequence plots for the mitochondiral genes: 

```
bsub -q long -R "rusage[mem=20G]" $BASE_DIR/software/diricore_subset/subsequence_analysis.sh 18436 hg19 5 all_excl_MT-transcripts
```

For RPF_density analysis the script is the same.

## 3. Plots_only

Sometimes (especially when plotting Mito subsequence results, the y_limits are out of range. You can simply add a fixed value after the plots_only option or you can use the option "max" and the script calculates the proper value

```
bsub -q long -R "rusage[mem=20G]" $BASE_DIR/software/diricore_subset/subsequence_analysis_MT.sh 18436 hg19 5 all_MT-transcripts all_samples plots_only max
```

```
bsub -q long -R "rusage[mem=20G]" $BASE_DIR/software/diricore_subset/subsequence_analysis.sh 18436 hg19 5 all_excl_MT-transcripts all_samples plots_only max
```
