# UCSC link

> **_IMPORTANT:_** `$BASE_DIR` has to be specified in your `.bash_profile`. Details [here](docs/0_before_you_start.md)

> **_IMPORTANT 2:_** The following tools are required and have to be placed into `$BASE_DIR/software/bin`:
> [bedtools v2.25.0](https://bedtools.readthedocs.io/en/latest/), [bedClip](http://hgdownload.cse.ucsc.edu/admin/exe/linux.x86_64/bedClip), [bedGraphToBigWig](http://hgdownload.cse.ucsc.edu/admin/exe/linux.x86_64/bedGraphToBigWig)


## 1. Create UCSC tracks

```
module load samtools
$BASE_DIR/software/ucsc/1_generate_ucsc_tracks.sh 21012 mm10 all_unique
```

This will output the following:

```
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S10_CoC_NG_LG.sh
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S1_E0771_HG.sh
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S2_E0771_LG.sh
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S3_E0771_LC.sh
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S4_CoC_GFP_HG.sh
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S5_CoC_GFP_LG.sh
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S6_MEF_HG.sh
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S7_MEF_LG.sh
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S8_MEF_LC.sh
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S9_CoC_NG_HG.sh
```

Copy & paste these commands to submit the jobs. 

## 2. Create annotation file

```
$BASE_DIR/software/ucsc/2_ucsc_track_annotation.sh 21012 mm10 all_unique
```

This will output the following:

```
Created file: /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/ucsc_track_annotation.txt
```

## 3. Upload tracks to the FTP

```
python $BASE_DIR/software/ucsc/3_ucsc_tracks_to_ftp.py 21012 all_unique
```

This will output the following:

```
2021-02-25 13:56:35
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S4_CoC_GFP_HG_minus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S2_E0771_LG_minus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S6_MEF_HG_plus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S9_CoC_NG_HG_plus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S6_MEF_HG_minus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S4_CoC_GFP_HG_plus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S9_CoC_NG_HG_minus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S5_CoC_GFP_LG_plus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S3_E0771_LC_minus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S1_E0771_HG_plus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S3_E0771_LC_plus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S7_MEF_LG_plus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S8_MEF_LC_plus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S10_CoC_NG_LG_plus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S1_E0771_HG_minus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S7_MEF_LG_minus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S2_E0771_LG_plus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S10_CoC_NG_LG_minus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S5_CoC_GFP_LG_minus.bw
Uploading /icgc/dkfzlsdf/analysis/OE0532/21012/analysis/output/gen_tracks/all_unique/S8_MEF_LC_minus.bw
```

> **_NOTE:_** DKFZ FTP-server stores files for a certain period of time, and then deletes them. 

## 4. Create UCSC session

This has to be done manually via UCSC web-site. 

Open the link: http://genome.ucsc.edu/cgi-bin/hgCustom

### 4.1. Delete existing tracks if there are any

![](/pics/ucsc_1.png)

### 4.2. Add custom tracks

![](/pics/ucsc_2.png)

### 4.3 Add tracks & select the genome version

Copy & paste the annotation file: `less $BASE_DIR/21012/analysis/output/gen_tracks/all_unique/ucsc_track_annotation.txt`

Make sure that the correct genome version is selected.

![](/pics/ucsc_3.png)

### 4.4. Open tracks in the UCSC browser

![](/pics/ucsc_4.png)

### 4.5 View the results

![](/pics/ucsc_5.png)

### 4.6. Save session

Open the UCSC sessions: http://genome.ucsc.edu/cgi-bin/hgSession

> **_NOTE_** You need to be registered, to be able to save sessions

Save the UCSC link: 

![](/pics/ucsc_6.png)

### 4.7 Share the UCSC link

Now the ucsc link can be shared: http://genome.ucsc.edu/s/stephz/21012_all_unique

## 5. IMPORTANT - keep cronjob running

> **_NOTE:_** DKFZ FTP-server stores files for a certain period of time, and then deletes them. 

It is important to regularly check that the files have not been removed. And if so, to upload them again.

This is done with the script: `python $BASE_DIR/software/ucsc/3_ucsc_tracks_to_ftp.py`

To let the script run automatically, set up a [crontab](https://crontab.guru/) with the command `crontab -e`

Add the following line (tab-separated):

```
21  6   *   *   *   python /omics/groups/OE0532/internal/from_snapshot/software/ucsc/3_ucsc_tracks_to_ftp.py
```

This means, every day at 6:21 am the script will check whether all UCSC tracks which are available on the cluster exist on FTP. If not, will upload them automatically. The user will be notified by email. 
