# Demultiplexing

> **_IMPORTANT:_** `$BASE_DIR` has to be specified in your `.bash_profile`. Details [here](/docs/0_before_you_start.md)

> **_IMPORTANT 2:_** `fastx_barcode_splitter.pl` has to be downloaded from [here](http://hannonlab.cshl.edu/fastx_toolkit/download.html) and placed into `$BASE_DIR/software/bin`

> **_IMPORTANT 3:_** [cutadapt](https://cutadapt.readthedocs.io/en/stable/) and [umi_tools](https://github.com/CGATOxford/UMI-tools) have to installed into p3 environment: 

> `pip install umi_tools` and `pip install cutadapt`


## Download the data

Let's take a dataset 20910 as an example. This is a mouse dataset.


1. When dataset is ready -> the notification will be sent by email
2. Download the data: `$BASE_DIR/software/preprocessing/0_get_data.sh 20910`
3. After that the dataset has been downloaded to: `$BASE_DIR/20910`

## Sequencing information
Now let's see what kind of dataset we received.
1. Open ILSE: `https://ilseform.dkfz.de/ilse-wui/secure/mySubmissions.jsf` and click on the dataset

We get a table which looks more or less like that: 

![table](/pics/ilse_table.png)

2. From that we know our Run ID: `210212_VH00211_26_AAAHM3LM5`. 

So we know that our raw data is here: `ls -lh $BASE_DIR/20910/210212_VH00211_26_AAAHM3LM5`

```
-rwxr-x---. 1 e984a B250 2.1K Feb 15 10:29 210212_VH00211_26_AAAHM3LM5_meta.tsv
drwxr-s---. 3 e984a B250   23 Feb 15 10:30 AS-580536-LR-55520
drwxr-s---. 3 e984a B250   23 Feb 15 10:28 AS-580538-LR-55520
drwxr-s---. 3 e984a B250   23 Feb 15 10:32 AS-580540-LR-55520
drwxr-s---. 3 e984a B250   23 Feb 15 10:29 Undetermined_1
```

3. We also can see the annotation for our `.fastq` files

`AS-580536-LR-55520_R1` is `24`

`AS-580538-LR-55520_R1` is `25`

`AS-580540-LR-55520_R1` is `26`

The same info we can get from the `meta.tsv` file which is located here: `$BASE_DIR/20910/20910_meta.tsv`

## Create symlinks
Before running the script, we need to put the files in the correct directory. Which is this one: `$BASE_DIR/20910/analysis/input/merged/`. The script will look for `.fastq` files in that directory.

So, let's create symlinks.
1. Symlink for the subset 24:  `ln -s $BASE_DIR/20910/210212_VH00211_26_AAAHM3LM5/AS-580536-LR-55520/fastq/AS-580536-LR-55520_R1.fastq.gz $BASE_DIR/20910/analysis/input/merged/24.fastq.gz`

2. Symlink for the subset 25: `ln -s $BASE_DIR/20910/210212_VH00211_26_AAAHM3LM5/AS-580538-LR-55520/fastq/AS-580538-LR-55520_R1.fastq.gz $BASE_DIR/20910/analysis/input/merged/25.fastq.gz`

3. Symlink for the subset 26: `ln -s $BASE_DIR/20910/210212_VH00211_26_AAAHM3LM5/AS-580540-LR-55520/fastq/AS-580540-LR-55520_R1.fastq.gz $BASE_DIR/20910/analysis/input/merged/26.fastq.gz` 

Now we can check that the symlinks have been created: `ls -lh $BASE_DIR/20910/analysis/input/merged`

```
total 120K
lrwxrwxrwx. 1 e984a B250 120 Feb 15 13:51 24.fastq.gz -> /icgc/dkfzlsdf/analysis/OE0532/20910/210212_VH00211_26_AAAHM3LM5/AS-580536-LR-55520/fastq/AS-580536-LR-55520_R1.fastq.gz
lrwxrwxrwx. 1 e984a B250 120 Feb 15 13:52 25.fastq.gz -> /icgc/dkfzlsdf/analysis/OE0532/20910/210212_VH00211_26_AAAHM3LM5/AS-580538-LR-55520/fastq/AS-580538-LR-55520_R1.fastq.gz
lrwxrwxrwx. 1 e984a B250 120 Feb 15 13:53 26.fastq.gz -> /icgc/dkfzlsdf/analysis/OE0532/20910/210212_VH00211_26_AAAHM3LM5/AS-580540-LR-55520/fastq/AS-580540-LR-55520_R1.fastq.gz 
```

## Create annotation files (bc_files)
To demultiplex the data, we need to create annotation files (bc_files). These files contain information about sample name and the barcode (tab separated). This info should be received from the person, who prepared the library for sequencing. 

> **_NOTE:_** The name of the `bc_file` should end with the name of the `.fastq` file, so that the script knows which bc file is for which fastq file. E.g. for the subset 24 the fastq file is named `24.fastq.gz` and bc file is named `bc_file_24.txt`

> **_SMALL HINT:_** Sample names should not start with a number! Some scripts (e.g. RiboWaltz) will fail! E.g. sample `231_3D_24h_AUF` is not a valid name. Example of a valid name: `S1_231_3D_24h_AUF`

1. bc_file for the subset 24: `vim $BASE_DIR/20910/analysis/input/metadata/bc_file_24.txt`

```
S5_T_Pool2    ATCGT
S6_Liver2    AGCTA
S7_Liver4    CGTAA
S8_Liver6    GATCA
S9_Liver8    GCATA
```

2. bc_file for the subset 25: `$BASE_DIR/20910/analysis/input/metadata/bc_file_25.txt`

```
S1_T2_late    AGCTA
S2_T4_late    CGTAA
S3_T5_late    CTAGA
S4_4T1_Ctrl    GATCA
```


3. bc_file for the subset 26: `$BASE_DIR/20910/analysis/input/metadata/bc_file_26.txt`

```
S10_T_Pool1    ATCGT
S11_Lung2    AGCTA
S12_Lung4    CGTAA
S13_Lung6    GATCA
S14_Lung8    GCATA
```

## Trim adapter and demultiplex
1. Trim and demultiplex the subset 24: `bsub -q long -R "rusage[mem=35G]" $BASE_DIR/software/preprocessing/1_trim_adapter_and_demultiplex.sh 20910 30 24`
2. Trim and demultiplex the subset 25: `bsub -q long -R "rusage[mem=35G]" $BASE_DIR/software/preprocessing/1_trim_adapter_and_demultiplex.sh 20910 30 25`
3. Trim and demultiplex the subset 26: `bsub -q long -R "rusage[mem=35G]" $BASE_DIR/software/preprocessing/1_trim_adapter_and_demultiplex.sh 20910 30 26`

where `30` is min read length, `24`, `25` and `26` are the subsets.

This will allow to process all 3 subsets in parallel.

> *_NOTE 1:_* The command `bsub` submits the job to the cluster. The job will be running for a few hours. Once it's done (or failed), the notification will be sent by email.
> 
> *_NOTE 2:_* If you want to repeat this step, be sure to remove the previous output:
> 
> rm -r $BASE_DIR/Proj_id/analysis/output/*
> 
> rm $BASE_DIR/Proj_id/analysis/input/fastq/*.gz
> 
> rm $BASE_DIR/Proj_id/analysis/input/merged/24_trimmed.fastq.gz


If the job has failed, the output will be included in the email. This might help to figure out what exactly went wrong.

## Results

As a result the following has been done:

1. Removed adapter sequence (with the tool [cutadapt](https://cutadapt.readthedocs.io/en/stable/))
2. Data demultiplexed (with [FASTX-toolkit barcode_splitter](http://hannonlab.cshl.edu/fastx_toolkit/commandline.html))
3. Extracted UMIs (with [umi_tools](https://github.com/CGATOxford/UMI-tools))
4. Created symlinks from extracted UMI files to `input/fastq` directory


###  Cutadapt output
1. Trimmed files. `ls -lh $BASE_DIR/20910/analysis/input/merged/*_trimmed.fastq.gz`

```
-rw-r--r--. 1 e984a B250 2.4G Feb 15 14:58 $BASE_DIR/20910/analysis/input/merged/24_trimmed.fastq.gz
-rw-r--r--. 1 e984a B250 2.2G Feb 15 17:17 $BASE_DIR/OE0532/20910/analysis/input/merged/25_trimmed.fastq.gz
-rw-r--r--. 1 e984a B250 286M Feb 15 18:29 $BASE_DIR/20910/analysis/input/merged/26_trimmed.fastq.gz
```

2. Stats files. `ls -lh $BASE_DIR/20910/analysis/output/*cutadapt_trimming_stats.txt`

```
-rw-r--r--. 1 e984a B250 4.0K Feb 15 14:58 /icgc/dkfzlsdf/analysis/OE0532/20910/analysis/output/24_cutadapt_trimming_stats.txt
-rw-r--r--. 1 e984a B250 4.2K Feb 15 17:17 /icgc/dkfzlsdf/analysis/OE0532/20910/analysis/output/25_cutadapt_trimming_stats.txt
-rw-r--r--. 1 e984a B250 3.4K Feb 15 18:29 /icgc/dkfzlsdf/analysis/OE0532/20910/analysis/output/26_cutadapt_trimming_stats.txt
```

### Fastx-toolkit barcode_splitter output
The demultiplexed samples will be located here: `ls -lh $BASE_DIR/20910/analysis/output/demultiplexed`

```
total 47G
-rw-r--r--. 1 e984a B250 1.2G Feb 15 18:34 dem_S10_T_Pool1.fastq
-rw-r--r--. 1 e984a B250 366M Feb 15 18:34 dem_S11_Lung2.fastq
-rw-r--r--. 1 e984a B250 247M Feb 15 18:34 dem_S12_Lung4.fastq
-rw-r--r--. 1 e984a B250 201M Feb 15 18:34 dem_S13_Lung6.fastq
-rw-r--r--. 1 e984a B250  96M Feb 15 18:34 dem_S14_Lung8.fastq
-rw-r--r--. 1 e984a B250 8.9G Feb 15 17:59 dem_S1_T2_late.fastq
-rw-r--r--. 1 e984a B250 4.1G Feb 15 17:59 dem_S2_T4_late.fastq
-rw-r--r--. 1 e984a B250 3.7G Feb 15 17:59 dem_S3_T5_late.fastq
-rw-r--r--. 1 e984a B250 129M Feb 15 17:59 dem_S4_4T1_Ctrl.fastq
-rw-r--r--. 1 e984a B250 7.1G Feb 15 15:23 dem_S5_T_Pool2.fastq
-rw-r--r--. 1 e984a B250 1.7G Feb 15 15:23 dem_S6_Liver2.fastq
-rw-r--r--. 1 e984a B250 2.5G Feb 15 15:23 dem_S7_Liver4.fastq
-rw-r--r--. 1 e984a B250 3.3G Feb 15 15:23 dem_S8_Liver6.fastq
-rw-r--r--. 1 e984a B250 3.6G Feb 15 15:23 dem_S9_Liver8.fastq
-rw-r--r--. 1 e984a B250 205M Feb 15 15:23 dem_unmatched_24.fastq
-rw-r--r--. 1 e984a B250 374M Feb 15 17:59 dem_unmatched_25.fastq
-rw-r--r--. 1 e984a B250  33M Feb 15 18:34 dem_unmatched_26.fastq
```

### UMI-tools output
UMI-tools `umi_tools extract` command extracts UMI (in our case 5nt long) from each read, and adds this information in the header. Then later, after alignment is done, the duplicate reads (containing the same UMI-sequence) can be removed with the command `umi_tools dedup`.

The script will output the files to `umi_extract` directory. Check it out: `ls -lh $BASE_DIR/20910/analysis/output/umi_extract`:

```
-rw-r--r--. 1 e984a B250 149M Feb 15 18:41 dem_S10_T_Pool1_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250  51M Feb 15 18:43 dem_S11_Lung2_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250  36M Feb 15 18:44 dem_S12_Lung4_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250  27M Feb 15 18:46 dem_S13_Lung6_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250  14M Feb 15 18:46 dem_S14_Lung8_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250 1.1G Feb 15 19:36 dem_S1_T2_late_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250 523M Feb 15 19:59 dem_S2_T4_late_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250 485M Feb 15 20:21 dem_S3_T5_late_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250  19M Feb 15 20:22 dem_S4_4T1_Ctrl_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250 885M Feb 15 21:06 dem_S5_T_Pool2_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250 221M Feb 15 21:17 dem_S6_Liver2_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250 331M Feb 15 21:34 dem_S7_Liver4_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250 414M Feb 15 21:55 dem_S8_Liver6_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250 439M Feb 15 22:15 dem_S9_Liver8_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250  30M Feb 15 22:17 dem_unmatched_24_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250  50M Feb 15 22:19 dem_unmatched_25_umi_extracted.fastq.gz
-rw-r--r--. 1 e984a B250 4.6M Feb 15 22:19 dem_unmatched_26_umi_extracted.fastq.gz
drwxr-sr-x. 2 e984a B250  612 Feb 15 22:19 logs
```

## Quality controls

### Cutadapt QC
As a cutadapt QC, we need to check that cutadapt found an adapter sequence (at least partially) in most of the reads. 
Let's take a look at the `trimming_stats` files: `less $BASE_DIR/20910/analysis/output/24_cutadapt_trimming_stats.txt`.

```
=== Summary ===

Total reads processed:             203,922,943
Reads with adapters:               199,854,862 (98.0%)
Reads that were too short:          54,099,538 (26.5%)
Reads written (passing filters):   145,755,324 (71.5%)
```

We can see that 98% of reads had an adapter sequence, which is perfect. 71,5% of the reads have been written. This is fine.

> **_NOTE:_** If the number of written reads is too low, we can increase this number by selecting shorter reads. Default min length is 30. To decrease min length, re-run the script with an additional parameter: `bsub -q long -R "rusage[mem=35G]" $BASE_DIR/software/preprocessing/1_trim_adapter_and_demultiplex.sh 20910 25 24`, where 25 is the min read length. 

> **_NOTE:_** it is NOT recommended to take reads shorter than 25. Because this length will be decreased even more in the next steps. 5nts will be removed as UMI, and 5nts will be removed as a barcode. So in the end we get a read with the length 15nts, which is not ideal for the alignment. 

### BC_split stats QC
BC split will create a `bc_split_stats.txt` file for each subset: `ls -lh $BASE_DIR/20910/analysis/output/*bc_split_stats.txt`

```
-rw-r--r--. 1 e984a B250 676 Feb 15 15:23 /icgc/dkfzlsdf/analysis/OE0532/20910/analysis/output/24_bc_split_stats.txt
-rw-r--r--. 1 e984a B250 577 Feb 15 17:59 /icgc/dkfzlsdf/analysis/OE0532/20910/analysis/output/25_bc_split_stats.txt
-rw-r--r--. 1 e984a B250 670 Feb 15 18:34 /icgc/dkfzlsdf/analysis/OE0532/20910/analysis/output/26_bc_split_stats.txt
```

Let's see how many reads per sample we get: `less $BASE_DIR/20910/analysis/output/24_bc_split_stats.txt`

```
Barcode Count   Location
S5_T_Pool2      57230138        /icgc/dkfzlsdf/analysis/OE0532/20910/analysis/output/demultiplexed/dem_S5_T_Pool2.fastq
S6_Liver2       13248222        /icgc/dkfzlsdf/analysis/OE0532/20910/analysis/output/demultiplexed/dem_S6_Liver2.fastq
S7_Liver4       19984778        /icgc/dkfzlsdf/analysis/OE0532/20910/analysis/output/demultiplexed/dem_S7_Liver4.fastq
S8_Liver6       25591654        /icgc/dkfzlsdf/analysis/OE0532/20910/analysis/output/demultiplexed/dem_S8_Liver6.fastq
S9_Liver8       28082363        /icgc/dkfzlsdf/analysis/OE0532/20910/analysis/output/demultiplexed/dem_S9_Liver8.fastq
unmatched       1618169 /icgc/dkfzlsdf/analysis/OE0532/20910/analysis/output/demultiplexed/dem_unmatched.fastq
total   145755324
```
This looks fine.

> **_NOTE:_** It is important to check how many reads were `unmatched`. This number should NOT be too high. Compare it with all samples. If one of the samples got too few reads, but we see a lot of reads in `unmatched` - it means (most likely) that the barcodes were messed up. 

## Plot stats
1. Extract the cutadapt stats from the cutadapt output:

```
$BASE_DIR/software/preprocessing/stats/subset_cutadapt_stats.sh 20910 24
$BASE_DIR/software/preprocessing/stats/subset_cutadapt_stats.sh 20910 25
$BASE_DIR/software/preprocessing/stats/subset_cutadapt_stats.sh 20910 26
```

###  Plot separately for each subset
Sometimes it is necessary to plot each subset separately in a separate plot. This can be done with the following command:

```
module load R/4.4.1-GCCcore-14.1.0

Rscript $BASE_DIR/software/preprocessing/stats/plot_basic_stats.r 20910 24
Rscript $BASE_DIR/software/preprocessing/stats/plot_basic_stats.r 20910 25
Rscript $BASE_DIR/software/preprocessing/stats/plot_basic_stats.r 20910 26
```

Alternatively, open the RStudio: `https://odcf-rstudio01.dkfz.de/` and run the script from there. 

As an output we get 3 cutadapt plots and 3 bc split plots: `ls -lh $BASE_DIR/20910/analysis/output/figures/`

```
total 400K
-rw-r--r--. 1 e984a B250 4.7K Feb 16 11:20 24_BCsplit_stats.pdf
-rw-r--r--. 1 e984a B250 4.6K Feb 16 11:19 24_Cutadapt_stats.pdf
-rw-r--r--. 1 e984a B250 4.6K Feb 16 11:23 25_BCsplit_stats.pdf
-rw-r--r--. 1 e984a B250 4.6K Feb 16 11:23 25_Cutadapt_stats.pdf
-rw-r--r--. 1 e984a B250 4.7K Feb 16 11:22 26_BCsplit_stats.pdf
-rw-r--r--. 1 e984a B250 4.6K Feb 16 11:22 26_Cutadapt_stats.pdf
```

### Plot cutadapt stats (all samples in one plot)

If we need to plot all samples together, we need to merge stats from all subsets into one file.

Merge cutadapt stats: ` $BASE_DIR/software/preprocessing/stats/multiple_cutadapt_stats.sh 20910`

Merge bc_split stats: ` $BASE_DIR/software/preprocessing/stats/multiple_bc_split_stats.sh 20910`

This will create 2 files:

```
 $BASE_DIR/20910/analysis/output/cutadapt_plot_stats.txt
 $BASE_DIR/20910/analysis/output/bc_split_stats.txt
```

Now, let's create a plot: 

```
module load R/4.4.1-GCCcore-14.1.0
Rscript $BASE_DIR/software/preprocessing/stats/plot_basic_stats.r 45280

```

Or, simply run it from the RStudio: `https://odcf-rstudio01.dkfz.de/`

### Copy data from the cluster

Copy data: `scp -r  USERNAME@odcf-worker01.dkfz.de:YOUR_BASE_DIR/20910/analysis/output/figures/* ~/analysis/20910/figures`

Note that `~/analysis/20910/figures` has to be created on your local computer. 

### The cutadapt plot

![table](/pics/cutadapt.png?v=4&s=20)

### The bc split plot

![table](/pics/bc_split.png?v=4&s=20)
