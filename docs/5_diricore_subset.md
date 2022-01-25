# Diricore on subset of genes

Let's consider the following list of Mitochondrial genes: 

```
MT-ATP8
MT-ATP6
MT-CO1
MT-CO2
MT-CO3
MT-CYB
MT-ND1
MT-ND2
MT-ND3
MT-ND4L
MT-ND4
MT-ND5
MT-ND6
```


## 0. Create a reference file - skip if already done

Transfer gene symbols to ENSEMBL transcript IDs via BioMart Tool from ENSEMBL.
Using 'Filter', expand 'Gene', enter your gene list. Choose HGNC symbols as external reference ID list input (e.g. 'MT-ND6).
Using 'Attributes', expand 'Gene' and check/uncheck the list output parameters.
Export and format the final list for the column of interest.

```
cat $BASE_DIR/static/hg19/MT-genes.txt | cut -d '|' -f 1 > $BASE_DIR/static/hg19/MT-transcripts.txt
```

## 1. Load samtools

```
module load samtools
```

## 2. Extract reads from bam: 

```
$BASE_DIR/software/diricore_subset/1_extract_bam.sh 22276 all $BASE_DIR/static/hg19/MT-transcripts.txt
```

New bam files will be written to: `$BASE_DIR/22276/analysis/output/diricore_subset/all_MT-transcripts/alignments/toGenome`

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
bsub -q long -R "rusage[mem=20G]" $BASE_DIR/software/diricore_subset/subsequence_analysis.sh 18436 hg19 5 all_MT-transcripts
```

### 2.2 Subsequence plots for the mitochondiral genes: 

```
bsub -q long -R "rusage[mem=20G]" $BASE_DIR/software/diricore_subset/subsequence_analysis_MT.sh 18436 hg19 5 all_excl_MT-transcripts
```

For RPF_density analysis the script is the same.
