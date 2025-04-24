# Periodicity (RiboWaltz)

> **_IMPORTANT:_** `$BASE_DIR` has to be specified in your `.bash_profile`. Details [here](docs/0_before_you_start.md)

> **_IMPORTANT 2:_** R Package [RiboWaltz](https://github.com/LabTranslationalArchitectomics/riboWaltz) is required. 

RiboWaltz requires bam files with **transcriptome** coordinates (not the genome!), e.g. 

```
VH00211:4:AAAKJH7M5:1:1308:66460:17414_GGTCTGATCA       256     ENST00000456328.2       227     0       32M     *       0       0       CTAGAGTGGGATGGGCCATTGTTCATCTTCTG    CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC        NH:i:12 HI:i:8
VH00211:4:AAAKJH7M5:1:2207:35917:30230_TCCTGGATCA       256     ENST00000456328.2       361     0       30M     *       0       0       TGTGTGGTGATGCCAGGCATGCCCTTCCCT  CCCCCCCCCCCCCCCCCCCCCCCCCCCCCC  NH:i:21 HI:i:17
VH00211:4:AAAKJH7M5:1:1410:39666:48214_TGGTGGATCA       256     ENST00000456328.2       660     0       30M     *       0       0       CACCCCGAGATCACATTTCTCACTGCCTTC  CCCCCCCCC;CCCCC;CCCC;CCCCCCCC;  NH:i:16 HI:i:4
VH00211:4:AAAKJH7M5:1:1308:66460:17414_GGTCTGATCA       256     ENST00000515242.2       224     0       32M     *       0       0       CTAGAGTGGGATGGGCCATTGTTCATCTTCTG
```

where `ENST00000456328.2` is the transcript_id (3rd column), and `227` is the coordinate (1-based) of the 5' end of the read within the transcript sequence (4th column).

If alignment is performed as described [here](/docs/5_align.md), these bam files will be created by STAR and written in this directory:

```
$BASE_DIR/22276/analysis/output/alignments/toTranscriptome
```

## 1. Create symlinks

```
/omics/groups/OE0532/internal/from_snapshot/software/ribo_waltz/1_create_symlinks.sh 20910 all_unique
```

This will create symlinks to the bam files in the following directory:

```
$BASE_DIR/20910/analysis/input/periodicity_bam/all_unique
```

This needs to be done, because RiboWaltz reads the whole directory, not a single bam file.
This will only be successful if deduplication was performed after alignment. For the usage of *all* reads symlink *BAM* files manually using *'ln -s'*.   

## 3. Run RiboWaltz
First, install riboWaltz using RStudio (eg. R/4.2.0) manually, plus necessary dependencies.
Then continue from the command line:

```
module load gcc/7.2.0
module load R/4.2.0
bsub -q long -R rusage[mem=40G] Rscript $BASE_DIR/software/ribo_waltz/2_periodicity_additional_pdf.r 20910 mm10 all_unique
```

## Modification
The prior modified script will also output the count tables that are the plotting input for the riboWaltz plots.
The output dir will be printed with the STDOUT. 

# Ribowaltz for a subset of samples

It can happen that the dataset has 2 pools, and we want to analyse each pool independently. 

Let's take the dataset 23108. We have 2 pools - p24_GFP and p25_HA

##  1. Create symlinks for a subset of samples

**Let's create symlinks for both subsets:**

```
$BASE_DIR/software/ribo_waltz/1_create_symlinks.sh 23108 all_unique p24_GFP
$BASE_DIR/software/ribo_waltz/1_create_symlinks.sh 23108 all_unique p25_HA
```

This step requires `rpf_density_samplenames` files in the `analysis/input/metadata` directory (see diricore analysis). In our case we need those 2 files:

```
$BASE_DIR/23108/analysis/input/metadata/rpf_density_samplenames_p24_GFP.tsv
$BASE_DIR/23108/analysis/input/metadata/rpf_density_samplenames_p25_HA.tsv
```

The script will check both files and will create symlinks in the following directories:

```
$BASE_DIR/23108/analysis/input/periodicity_bam/all_unique_p24_GFP
$BASE_DIR/23108/analysis/input/periodicity_bam/all_unique_p25_HA
```

With this, we can run RiboWaltz

## 2. Run RiboWaltz


```
module load gcc/7.2.0
module load R/3.6.2
bsub -q long -R "rusage[mem=50G]" Rscript /omics/groups/OE0532/internal/Alex//scripts/2_periodicity_pdf.r 23108 hg19 all_unique_p24_GFP
bsub -q long -R "rusage[mem=50G]" Rscript /omics/groups/OE0532/internal/Alex//scripts/2_periodicity_pdf.r 23108 hg19 all_unique_p25_HA
```

## 2. psites per transcript

To obtain information and plotting about psite distribution per transcript, perform:

```
module load gcc/7.2.0
module load R/3.6.2
bsub -q long -R "rusage[mem=50G]" Rscript /omics/groups/OE0532/internal/Alex//scripts/2_periodicity_per_transcript.r 23108 hg19 all_unique_p24_GFP ENST00000229239.5
```
Important is the command line parameter of a ENSEMBL transcript ID version (here: ENST00000229239.5). Useful for single runs, however, iteration over multiple IDs should be performed via Rstudio.
That is, every iteration of Ribowaltz loads BAM files and calculates p-site offsets. Thus, to save time iterate these IDs from the script AFTER p-site offset calculation. 










