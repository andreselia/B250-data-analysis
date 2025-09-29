# Alignment

The alignment is performed with the tool [STAR](https://github.com/alexdobin/STAR)

The tool requires 2 reference files: 
1. Genome sequences (*.fasta)
2. Annotation file (*.gtf)

The reference files have been downloaded from the `gencode`: https://www.gencodegenes.org/

We use the reference genome [hg19](https://www.gencodegenes.org/human/release_19.html) (diricore supports only `hg19` for human and `mm10` for mouse)

## Build STAR index. 
### UPDATE 2025: This step was already done for version 2.5.3a. The specific version is now available on the Debian module list. 

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

`module load STAR/2.5.3a-GCC-14.1.0`
`module load SAMtools/1.20-GCC-14.1.0`

Align: `$BASE_DIR/software/preprocessing/3_alignment/1_align_to_transcriptome.sh 20910 mm10 80G`
### UPDATE 2025: The previous code prepares the sh file for this analysis. I commented (#) the part that loads STAR and Samtools since the versions may be slightly different. 
So, modules should be loaded before running this step.

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
...
```

Just copy & paste these commands to submit the jobs to the cluster.

2. Deduplicate transcriptome: `$BASE_DIR/software/preprocessing/3_alignment/2_deduplicate_umi.sh 20910 all --trans`

These will output the following: 

```
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S10_T_Pool1_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S11_Lung2_toTranscriptome.bam.sh
bsub -q long -R "rusage[mem=10G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/20910/deduplicating/S12_Lung4_toTranscriptome.bam.sh
...
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
...
```

`*_toTranscriptome.bam` files contain all reads (including duplicates)

`*_toTranscriptome_dedup.bam` files contain only unique reads (without duplicates)

`*_toTranscriptome.bam.bai` files are the index files which are required for deduplicating step and created automatically by the script. 


### Alignment stats

#### 0. Load samtools

To get the alignment stats, [samtools](http://quinlanlab.org/tutorials/samtools/samtools.html) is required. 

On the cluster the samtools can be loaded with this command: 

```
module load SAMtools/1.20-GCC-14.1.0
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
module load R/4.4.3-GCCcore-14.1.0
Rscript /omics/groups/OE0532/internal/Andres/scripts/scripts/plot_star_alignment_stats_new.r 44223
```

The alignments plot looks like that: 

![table](/pics/alignment_stats.png)
