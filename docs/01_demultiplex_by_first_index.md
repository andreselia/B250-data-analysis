# Demultiplex by first index

Sometimes we receive datasets which have not been split into pools.

There are 2 options:

1. Write to the facility and ask them to demultiplex (may take a while).

2. Demultiplex it ourselves. 

As an example, let's take the dataset 23108

## 1. Create bc_file

Create bc_file: `vim $BASE_DIR/23108/analysis/input/metadata/bc_file_first_index.txt` 

> **_IMPORTANT:_** bc_file has to be named exactly like this: `bc_file_first_index.txt`

```
p24_GFP ATCACG
p25_HA  CGATGT
```

## 2. Demultiplex

For this we use cutadapt. Yes, we treat our barcodes as adapter sequences. Cutadapt will look for the barcodes somewhere in the 3' end of each read, trim this sequence and discard the reads which do not contain this barcode.

The script requires 2 parameters:

1. `project_id`, which is `23108`
2. `path_to_fastq_file`, which in our case is `$BASE_DIR/23108/210907_VH00211_102_AAAM75GM5/AS-691001-LR-58000/fastq/AS-691001-LR-58000_R1.fastq.gz`

The .fastq file is usually located in the project directory, in the directory which is named as a run_id. For each dataset this name will be different, so just check what is in the dir `$BASE_DIR/$project_id`

**Run script**: 

```
$BASE_DIR/software/preprocessing/0_demultiplex_first_index.sh 23108 $BASE_DIR/23108/210907_VH00211_102_AAAM75GM5/AS-691001-LR-58000/fastq/AS-691001-LR-58000_R1.fastq.gz
```

This will create 2 scripts and output the following:

```
bsub -q long -R "rusage[mem=20G]" /omics/groups/OE0532/internal/from_snapshot//tmp/23108/first_index/p24_GFP.sh
bsub -q long -R "rusage[mem=20G]" /omics/groups/OE0532/internal/from_snapshot//tmp/23108/first_index/p25_HA.sh
```

Run those 2 scripts. 

The demultiplexed pools will be written in this directory: 

```
$BASE_DIR/23108/analysis/input/merged/p24_GFP.fastq.gz
$BASE_DIR/23108/analysis/input/merged/p25_HA.fastq.gz
```

To see how many reads each file got, let's look at the stats:

```
$BASE_DIR/23108/analysis/output/p24_GFP_cutadapt_trimming_stats.txt
$BASE_DIR/23108/analysis/output/p25_HA_cutadapt_trimming_stats.txt
```

That's all. Now we can proceed with the standard demultiplexing.
