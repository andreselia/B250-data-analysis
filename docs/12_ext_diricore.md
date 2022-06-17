# Ext Diricore (P-site signalling)

> **_IMPORTANT:_** This step requires [plastid](https://plastid.readthedocs.io/en/latest/). To install, run the command: `pip install plastid` 

## 0. Set up plastid

Install plastid: 

```
source activate p3
pip install plastid
```


Create reference for human (for mouse it's the same, just take the annotation for mouse - replace `hg19` with `mm10`): 

```
bsub -q long -R "rusage[mem=50G]" metagene generate $BASE_DIR/static/hg19/plastid --landmark cds_start --annotation_files $BASE_DIR/static/hg19/gencode.annotation.gtf --downstream 200
```

This will create 2 files `plastid_rois.bed` and `plastid_rois.txt` at the static directory: `$BASE_DIR/static/hg19/`



## 1. Get sequences from bam

Get sequences: 

```
module load samtools
bsub -R "rusage[mem=10G]" $BASE_DIR/software/ext_diricore/1_get_seq_from_bam.sh 22276 all_unique
```

## 2. Calculate offsets by plastid 

**Calculate offsets:**

```
$BASE_DIR/software/ext_diricore/2_get_offset.sh 22276 all_unique hg19
``` 

This will calculate the offset for each read length (per sample), and create some images + data files in this directory: `$BASE_DIR/22276/analysis/output/plastid/p_offsets/`

Copy the plots from the cluster (run it locally on the laptop): 

```
scp -r USERNAME@odcf-cn34u03s10:BASE_DIR/22276/analysis/output/figures/p_offsets ~/analysis/22276/figures/
```

`USERNAME` has to be replaced with the real username (e.g. `e984a`), and `BASE_DIR` has to be replaced with the real BASE_DIR (e.g. `/omics/groups/OE0532/internal/from_snapshot/`)

The plot will look like that:

![](/pics/plastid.png) 

## 3. Re-format offset files

Now we need to extract the offset and eliminate non-relevant data from the plastid output.

Re-format offset files:

```
$BASE_DIR/software/ext_diricore/3_reformat_offset.sh 22276 all_unique
```

The script will remove lines which we don't need, so that the final offset file will look like that:

```
length  p_offset
25      7
26      10
27      9
28      10
29      11
30      12
31      13
32      14
33      5
34      15
35      13
```

## 4. Manually adjust offset files

So we have to remove the lines which don't look that good. In this case, the final result will look like that:

```
length  p_offset
28  10
29  11
30  12
31  13
32  14
```

How do we know which ones look good and which ones don't? Generally, the perfect lengths are around 28-30 and perfect offset is around 12. If we see that the offset is too low (e.g. 7 or 5) or too high (e.g. 20 or 50) - discard these rows. 

In this case the length 33 has offset 5 - this looks wierd, so I remove everything after 33. 

Also, the default value is 13, so if we see too many 13s - most likely something is wrong. 

Repeat this for all samples.

## 5. Get normalized counts 

```
bsub -q long -R "rusage[mem=20G]" python $BASE_DIR/software/counts/2_normalize_counts.py 22276 all_unique hg19
```

Output dir: `$BASE_DIR/22276/analysis/output/alignments/reads_per_gene/normalized/`

## 6. Get psites counts 

```
bsub -q long -R "rusage[mem=20G]" python $BASE_DIR/software/ext_diricore/4_psites_counts.py 22276 all_unique 20 hg19
```

Output dir1: `$BASE_DIR/22276/analysis/output/ext_diricore/all_unique/psites`

Output dir2: `$BASE_DIR/22276/analysis/output/ext_diricore/all_unique/psites_dotplot`

## 7. Plot results

The psite counts have been calculated with 3 types of normalization: CPM, TPM and RPKM. We can choose which normalization we use in the plots (tpm is default)


### Plot p-sites, a-sites and e-sites per sample 

TPM:

```
python $BASE_DIR/software/ext_diricore/5_psites_plot.py 22276 all_unique
```

or 

```
python $BASE_DIR/software/ext_diricore/5_psites_plot.py 22276 all_unique tpm
```

RPKM: 

```
python $BASE_DIR/software/ext_diricore/5_psites_plot.py 22276 all_unique rpkm
```

This will produce 2 types of plots - grouped by codon and grouped by amino acid

Grouped by amino acid:

![](/pics/ext_diricore_1.png)

Grouped by codon:

![](/pics/ext_diricore_2.png)

### Plot p-sites, a-sites and e-sites per contrast

TPM: 

```
python $BASE_DIR/software/ext_diricore/5_psites_plot_contrasts.py 22276 all_unique
```

RPKM: 

```
python $BASE_DIR/software/ext_diricore/5_psites_plot_contrasts.py 22276 all_unique rpkm
``` 

Grouped by amino acid:

![](/pics/ext_diricore_3.png)

Grouped by codon:

![](/pics/ext_diricore_4.png)


### Plot reads per frame

```
module load R/3.6.2 && module load gcc/7.2.0 && Rscript $BASE_DIR/software/ext_diricore/5_plot_reads_per_frame.r 22276 all_unique
```

¡[](/pics/ext_diricore_5.png)

### Web-site results

Upload to the web-site: `python $BASE_DIR/software/project_list.py psites 22276 all_unique --remote` 

See the results: http://b250-bioinfo:5000/psite_plot/22276

![](/pics/ext_diricore_6.png)


# Ext diricore for a subset of genes

Let's say we want to perform this analysis only on mitochondrial genes.

## 1. Get sequences from bam: 

```
bsub -R "rusage[mem=10G]" $BASE_DIR/software/ext_diricore/1_get_seq_from_bam.sh 22276 all_unique $BASE_DIR/static/hg19/MT-transcripts.txt
```

The output will be: `$BASE_DIR/22276/analysis/output/ext_diricore/all_unique_MT-transcripts/tsv/`

For convenience, I rename this dir: 

```
mv $BASE_DIR/22276/analysis/output/ext_diricore/all_unique_MT-transcripts $BASE_DIR/22276/analysis/output/ext_diricore/all_unique_MT-genes
```

## 2. Create bam files with MT-genes only

1. Get seq IDs from transcriptome bams
```
mkdir -p $BASE_DIR/22276/analysis/output/diricore_subset/bam_ids/all_unique-MT-genes/
for f in $(ls $BASE_DIR/22276/analysis/output/alignments/toTranscriptome/*_toTranscriptome.bam); 
   do fn=$(basename $f); 
   fn=${fn%_toTranscriptome.bam}; 
   echo "samtools view $f | fgrep -w -f $BASE_DIR/static/hg19/MT-transcripts.txt | cut -f 1 > $BASE_DIR/22276/analysis/output/diricore_subset/bam_ids/all_unique-MT-genes/${fn}.txt"; done
```

2. Create sam headers 

```
mkdir -p $BASE_DIR/22276/analysis/output/diricore_subset/all_unique_MT-genes/sam/`
for f in $(ls $BASE_DIR/22276/analysis/output/diricore_subset/bam_ids/all_unique_MT-genes/*txt); 
  do fn=$(basename $f); 
  fn=${fn%.txt}; 
  echo "samtools view -H $BASE_DIR/22276/analysis/output/alignments/toGenome/${fn}_toGenome_dedup.bam > $BASE_DIR/22276/analysis/output/diricore_subset/all_unique_MT-genes/sam/${fn}.sam"; 
done
```

3. Extract reads from genome bams: 

```
for f in $(ls $BASE_DIR/22276/analysis/output/diricore_subset/bam_ids/all_unique_MT-genes/*txt); 
    do fn=$(basename $f); 
    fn=${fn%.txt}; 
    echo "samtools view $BASE_DIR/22276/analysis/output/alignments/toGenome/${fn}_toGenome.bam | fgrep -w -f $f >>   $BASE_DIR/22276/analysis/output/diricore_subset/all_unique_MT-genes/sam/${fn}.sam"; 
done
```

4. Convert sam to bam:

```
mkdir -p $BASE_DIR/22276/analysis/output/diricore_subset/all_unique_MT-genes/alignments/toGenome/

for f in $(ls $BASE_DIR/22276/analysis/output/diricore_subset/all_unique_MT-genes/sam/*.sam); 
    do fn=$(basename $f); 
    fn=${fn%.sam}; 
    echo "samtools view -b $f > $BASE_DIR/22276/analysis/output/diricore_subset/all_unique_MT-genes/alignments/toGenome/${fn}_toGenome.bam"; 
done
```

## 3. Psite analysis

From here just perform the steps from the original [Ext diricore analysis](#ext-diricore-p-site-signalling)

After that, all intermediate files can be removed:

```
rm -rf $BASE_DIR/22276/analysis/output/diricore_subset/all_unique_MT-genes/
rm -rf $BASE_DIR/22276/analysis/output/diricore_subset/bam_ids/all_unique-MT-genes/
```
