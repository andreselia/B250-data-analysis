# UCSC link

> **_IMPORTANT:_** `$BASE_DIR` has to be specified in your `.bash_profile`. Details [here](docs/0_before_you_start.md)

> **_IMPORTANT 2:_** The following tools are required and have to be placed into `$BASE_DIR/software/bin`:
> [bedtools v2.25.0](https://bedtools.readthedocs.io/en/latest/), [bedClip](http://hgdownload.cse.ucsc.edu/admin/exe/linux.x86_64/bedClip), [bedGraphToBigWig](http://hgdownload.cse.ucsc.edu/admin/exe/linux.x86_64/bedGraphToBigWig)

> The tools are already installed, but bedGraphToWig requires libpng12, outdated in Debian. I reinstalled in /software2/bin and now I am modifing $BASE_DIR/software/ucsc/bdg2bw.sh to call BIN_PATH2=$BASE_DIR/software2/bin folder to check if the reinstallation fix the issues
> The reinstallation fixed the issues, and I am keeping the script to its original form $BASE_DIR/software/ucsc/bdg2bw.sh
> **_NOTE:_** DKFZ FTP-server is no longer available. So the files are "permanently" stored in Nextcloud. Cronjob is no longer necessary 

## 1. Create UCSC tracks

```
module load SAMtools/1.20-GCC-14.1.0
module load BEDTools/2.31.1-GCC-14.1.0

$BASE_DIR/software/ucsc/1_generate_ucsc_tracks.sh 21012 mm10 all_unique
```

This will output the following:

```
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S10_CoC_NG_LG.sh
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S1_E0771_HG.sh
bsub -q long -R "rusage[mem=30G]" /icgc/dkfzlsdf/analysis/OE0532/tmp/21012/ucsc/S2_E0771_LG.sh
...
```

Copy & paste these commands to submit the jobs. 

## 2. Upload .bw files to Nextcloud and create an annotation file

```
$BASE_DIR/software/ucsc/2_ucsc_track_nextcloud_and_annotation.sh 44312_HCT a690a pass nextcloud_id
```

This will output the following:

```
Results successfully saved to: /omics/groups/OE0532/internal/Andres//44312_HCT/analysis/output/ucsc_tracks/all/ucsc_tracks_listing.txt
```
You can do "less" to read the annotation file or copy-paste the output from the terminal

## 4. Create UCSC session

This has to be done manually via the UCSC website. 

Open the link: http://genome.ucsc.edu/cgi-bin/hgCustom

### 4.1. Delete existing tracks if there are any

![](/pics/ucsc_1.png)

### 4.2. Add custom tracks

![](/pics/ucsc_2.png)

### 4.3 Add tracks & select the genome version

Copy & paste the annotation file: `less /omics/groups/OE0532/internal/Andres//44312_HCT/analysis/output/ucsc_tracks/all/ucsc_tracks_listing.txt`

Make sure that the correct genome version is selected.

![](/pics/ucsc_3.png)

### 4.4. Open tracks in the UCSC browser

Click 'Go'

![](/pics/ucsc_4.png)

### 4.5 View the results

![](/pics/ucsc_5.png)

### 4.6. Save session

Open the UCSC sessions: http://genome.ucsc.edu/cgi-bin/hgSession

> **_NOTE_** You need to be registered, to be able to save sessions

Save the UCSC link: 

![](/pics/ucsc_6.png)

### 4.7 Share the UCSC link

Now the ucsc link can be shared: http://genome.ucsc.edu/s/andres.elia/44312_Hum_all

## 5. IMPORTANT - about cronjob 

> **_NOTE:_** DKFZ FTP-server is no longer available. So the files are "permanently" stored in Nextcloud. Cronjob is no longer necessary 

