# Alignment

The alignment is performed with the tool [STAR](https://github.com/alexdobin/STAR)

The tool requires 2 reference files: 
1. Genome sequences (*.fasta)
2. Annotation file (*.gtf)

The reference files have been downloaded from the `gencode`: https://www.gencodegenes.org/

We use the reference genome [hg19](https://www.gencodegenes.org/human/release_19.html) (diricore supports only `hg19` for human and `mm10` for mouse)

## Build STAR index (skip this step if aready done)

1. Download genome sequences: `Genome sequence (GRCh37.p13)`
```
curl ftp://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_19/GRCh37.p13.genome.fa.gz $BASE_DIR/static/hg19/GRCh37.p13.genome.fa
```

2. Download annotation file: `Comprehensive gene annotation` 

```
curl ftp://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_19/gencode.v19.annotation.gtf.gz $BASE_DIR/hg19/gencode.v19.annotation.gtf
```

3. Build STAR index 

```
bsub -q long -R "rusage[mem=100G]" STAR --runThreadN 4 --runMode genomeGenerate 
    --genomeDir $BASE_DIR/static/hg19 
    --genomeFastaFiles $BASE_DIR/hg19/GRCh37.p13.genome.fa 
    --sjdbGTFfile $BASE_DIR/static/hg19/gencode.v19.annotation.gtf --sjdbOverhang 100
```

## Alignment

`module load STAR`

Align: `$BASE_DIR/software/preprocessing/3_alignment/1_align_to_transcriptome.sh 20910 mm10 80G`

The script takes input file from the clean directory: `$BASE_DIR/20910/analysis/output/clean`

* `20910` is the name of the dataset

* `mm10` is the genome 

* `80G` is the memory which has to be reserved to submit the job on the cluster (default is 40G).

STAR outputs 2 types of bam files into the following directories: 

* `$BASE_DIR/20910/analysis/output/alignments/toGenome`
* `$BASE_DIR/20910/analysis/output/alignments/toTranscriptome`

`toGenome` directory contains bam files with the genome-based coordinates, e.g. `chr1 pos 9877`. These bam files will be used for the further analysis, such as `diricore`

`toTranscriptome` directory contains bam files with the transcriptome-based coordinates, e.g. `ENSMUST00000162897 pos 973`. These bam files will be used for `RiboWaltz` and some other analysis. 

## Deduplicating

After the alignment is done, let's get rid of the duplicated reads (PCR artifacts)

1. Deduplicate genome: `$BASE_DIR/software/preprocessing/3_alignment/2_deduplicate_umi.sh 20910 all`

This will output the following: 

```
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S10_T_Pool1_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S11_Lung2_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S12_Lung4_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S13_Lung6_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S14_Lung8_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S1_T2_late_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S2_T4_late_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S3_T5_late_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S4_4T1_Ctrl_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S5_T_Pool2_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S6_Liver2_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S7_Liver4_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S8_Liver6_toGenome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S9_Liver8_toGenome.bam.sh
```

Just copy & paste these commands to submit the jobs to the cluster.

2. Deduplicate transcriptome: `$BASE_DIR/software/preprocessing/3_alignment/2_deduplicate_umi.sh 20910 all --trans`

These will output the following: 

```
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S10_T_Pool1_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S11_Lung2_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S12_Lung4_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S13_Lung6_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S14_Lung8_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S1_T2_late_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S2_T4_late_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S3_T5_late_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S4_4T1_Ctrl_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S5_T_Pool2_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S6_Liver2_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S7_Liver4_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S8_Liver6_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S9_Liver8_toTranscriptome.bam.sh
```

> **_IMPORTANT:_** It is safer to run transcriptome and genome deduplicating one after another (NOT at the same time!). STAR, when processing 2 different files with the same names, can mix them up! 

In each directory `toGenome` and `toTranscriptome` we can see the following files: 

```
-rw-r--r--. 1 e984a B250 197M Feb 16 16:22 S1_T2_late_toTranscriptome.bam
-rw-r--r--. 1 e984a B250 7.3M Feb 16 16:56 S1_T2_late_toTranscriptome.bam.bai
-rw-r--r--. 1 e984a B250  51M Feb 16 17:02 S1_T2_late_toTranscriptome_dedup.bam
-rw-r--r--. 1 e984a B250 280M Feb 16 16:22 S2_T4_late_toTranscriptome.bam
-rw-r--r--. 1 e984a B250 7.5M Feb 16 16:56 S2_T4_late_toTranscriptome.bam.bai
-rw-r--r--. 1 e984a B250  78M Feb 16 17:10 S2_T4_late_toTranscriptome_dedup.bam
-rw-r--r--. 1 e984a B250 224M Feb 16 16:21 S3_T5_late_toTranscriptome.bam
-rw-r--r--. 1 e984a B250 7.1M Feb 16 16:56 S3_T5_late_toTranscriptome.bam.bai
-rw-r--r--. 1 e984a B250  55M Feb 16 17:08 S3_T5_late_toTranscriptome_dedup.bam
-rw-r--r--. 1 e984a B250 8.6M Feb 16 16:12 S4_4T1_Ctrl_toTranscriptome.bam
-rw-r--r--. 1 e984a B250 3.7M Feb 16 16:55 S4_4T1_Ctrl_toTranscriptome.bam.bai
-rw-r--r--. 1 e984a B250 4.8M Feb 16 16:57 S4_4T1_Ctrl_toTranscriptome_dedup.bam
-rw-r--r--. 1 e984a B250 371M Feb 16 16:31 S5_T_Pool2_toTranscriptome.bam
-rw-r--r--. 1 e984a B250 7.3M Feb 16 16:56 S5_T_Pool2_toTranscriptome.bam.bai
-rw-r--r--. 1 e984a B250  66M Feb 16 17:06 S5_T_Pool2_toTranscriptome_dedup.bam
```

`*_toTranscriptome.bam` files contain all reads (including duplicates)

`*_toTranscriptome_dedup.bam` files contain only unique reads (without duplicates)

`*_toTranscriptome.bam.bai` files are the index files which are required for deduplicating step and created automatically by the script. 


### Alignment stats

#### 0. Load samtools

To get the alignment stats, [samtools](http://quinlanlab.org/tutorials/samtools/samtools.html) is required. 

On the cluster the samtools can be loaded with this command: 

```
module load samtools
```

#### 1. Get alignment stats 

```
bsub -q medium -R "rusage[mem=30G]" $BASE_DIR/software/preprocessing/stats/get_alignment_stats.sh 20910
```

#### 2. Check bc_split_stats

This step requires `bc_split_stats.txt`. If dataset contains several pools, most likely each pool has it's own bc_split_stats file.

To aggregate all bc_split stats into a single file, do 

```
$BASE_DIR/software/preprocessing/stats/multiple_bc_split_stats.sh 20910
```

Output file will be located here: `$BASE_DIR//20910/analysis/output/bc_split_stats.txt`


#### 3. Aggregate stats into 1 file 

```
python $BASE_DIR/software/preprocessing/stats/get_alignment_stats_2.py 20910
```

#### 4. Plot alignment stats

```
module load gcc/7.2.0 && module load R/3.6.2 && Rscript /omics/groups/OE0532/internal/Alex/scripts/plot_star_alignment_stats_new.r 20910
```

The alignments plot looks like that: 

![table](/pics/alignment_stats.png)
