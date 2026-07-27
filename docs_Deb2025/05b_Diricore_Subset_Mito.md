<img width="442" height="58" alt="image" src="https://github.com/user-attachments/assets/f53ef213-106b-4bef-b9fc-c7976f465414" /># Diricore on subset of genes

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

```
vim $BASE_DIR/46700/analysis/output/Mito-transcripts.txt
```

## 1. Load samtools

```
module load SAMtools/1.20-GCC-14.1.0 

```

## 2. Extract reads from bam: 

This new script will create a bed file with genomic coordinates, and a list of scripts to bu run in order to extract the reads. Also it will provide the number of reads for each gene.

```
bsub -q medium -R "rusage[mem=50G]" $BASE_DIR/software/diricore_subset/1_extract_bam_v4.sh 46700 all $BASE_DIR/46700/analysis/output/MT-transcripts.txt
```

New bam files will be written to: `$BASE_DIR/22276/analysis/output/diricore_subset/all_MT-transcripts/alignments/toGenome`
For Mito genes, the bed file needs to be fixed with

```
sed 's/^chrMT/chrM/' /omics/groups/OE0532/internal/Andres//tmp/ext_diricore/46700/Mito-transcripts.bed > /omics/groups/OE0532/internal/Andres//tmp/ext_diricore/46700/MT-transcripts.bed
```

## 3. Run rpf density analysis: 

```
bsub -q long -R "rusage[mem=20G]" $BASE_DIR/software/diricore_subset/rpf_density_analysis.sh 18436 hg19 5 all_MT-transcripts
```

## 4. Subsequence analysis:

```
bsub -q long -R "rusage[mem=20G]" $BASE_DIR/software/diricore_subset/subsequence_analysis_MT.sh 18436 hg19 5 all_MT-transcripts
```

As a QC analysis, you can run a test to get a table with read length if they are in frame 0 1 or 2 and also the plots. Don´t forget to target the index table. If you are running another subset of genes, omit this option.
Results can be found in .../diricore_subset/<subset>/length_frame_qc/


```
bsub -q long -R "rusage[mem=50G]" $BASE_DIR/scripts/scripts_andres/length_frame_distribution.sh 46700 hg19 all_MT-transcripts subseq_index_data_MT.pkl.gz

scp -r -p a690a@odcf-worker01.dkfz.de:/omics/groups/OE0532/internal/Andres//46700/analysis/output/diricore_subset/all_MT-transcripts/length_frame_qc ~/analysis/46700/

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

### 2.1 Subsequence plots for the mitochondrial genes: 

```
bsub -q long -R "rusage[mem=20G]" $BASE_DIR/software/diricore_subset/subsequence_analysis_MT.sh 18436 hg19 5 all_MT-transcripts
```

### 2.2 Subsequence plots for the cytosolic genes: 

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
