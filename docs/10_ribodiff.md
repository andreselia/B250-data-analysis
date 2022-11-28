# RiboDiff (Translational Efficiency)

> **_IMPORTANT:_** `$BASE_DIR` has to be specified in your `.bash_profile`. Details [here](docs/0_before_you_start.md)

> **_IMPORTANT 2:_** This analysis will work for the datasets which have replicates. 

> **_IMPORTANT 3:_** [RiboDiff](https://github.com/kate-v-stepanova/RiboDiff) has to be installed!

This analysis is performed for coupled RNA-seq and RP datasets.

It requires replicates. If no replicates, then split into replicates: 

```
$BASE_DIR/software/RiboDiff/b250_scripts/split_into_replicates.py PROJECT_ID
```

(maybe this script needs some modifications)

## 0. Install RiboDiff - skip if already done!

This requires python2.7!

Clone repo into `software` dir:

```
git clone git@github.com:kate-v-stepanova/RiboDiff
```

Activate p2 virtual env:

```
conda activate diricore
```

Install RiboDiff

```
cd RiboDiff
python setup.py build
python setup.py test (optional step)
python setup.py install
```

For more installation instructions, check the file: `$BASE_DIR/software/RiboDiff/INSTALL`

## 1. Get sequences from bam

RNA-seq:
```
bsub -q long -R "rusage[mem=30G]" $BASE_DIR/software/ext_diricore/1_get_seq_from_bam.sh 21221_RNA all_unique
```
Use 'long' processing time for RNAseq data!.

If for RNA-seq data there are no UMIs, then the script will not find the `*dedup.bam` files. Just run the script with the argument `all`. And then create symlinks:

```
mkdir -p $BASE_DIR/21221_RNA/analysis/output/ext_diricore/all_unique/tsv/
for f in $(ls $BASE_DIR/21221_RNA/analysis/output/ext_diricore/all/tsv/*.tsv); do fn=$(basename $f); echo "ln -s $f $BASE_DIR/21221_RNA/analysis/output/ext_diricore/all_unique/tsv/$fn"; done
```

RP: 

```
bsub -q medium -R "rusage[mem=30G]" $BASE_DIR/software/ext_diricore/1_get_seq_from_bam.sh 21221 all_unique
```

> **_Note:_** Samplenames in both datasets have to MATCH! If not, please rename.


## 2. Aggregate counts 

coding genes: 

human
```
while read -r contrast; do echo "bsub -q medium -R \"rusage[mem=30G]\" python $BASE_DIR/software/ribo_diff/aggregate_counts2.py 21221_RNA 21221 all_unique $contrast"; done < $BASE_DIR/21221/analysis/input/metadata/rpf_density_contrasts.tsv
```
mouse script (combine with upper while command)
```
Alex/scripts/aggregate_counts2_mouse.py
```

## 3. Create metafiles

Create dir: 

```
mkdir -p $BASE_DIR/21221/analysis/input/ribo_diff/
```

Touch metafiles:

```
while read -r contrast; do s=$(echo $contrast | cut -f1 -d-); c=$(echo $contrast | cut -f2 -d-);  echo "touch $BASE_DIR/21221/analysis/input/ribo_diff/${s}_vs_${c}.metafile.csv"; done < $BASE_DIR/21221/analysis/input/metadata/rpf_density_contrasts.tsv
```

The output will be: (copy & paste it!)

```
touch /icgc/dkfzlsdf/analysis/OE0532/21221/analysis/input/ribo_diff/GFP_sh_vs_Ctrl_sh.metafile.csv
touch /icgc/dkfzlsdf/analysis/OE0532/21221/analysis/input/ribo_diff/sh12_vs_Ctrl_sh.metafile.csv
touch /icgc/dkfzlsdf/analysis/OE0532/21221/analysis/input/ribo_diff/sh40_vs_Ctrl_sh.metafile.csv
touch /icgc/dkfzlsdf/analysis/OE0532/21221/analysis/input/ribo_diff/sh12_vs_GFP_sh.metafile.csv
touch /icgc/dkfzlsdf/analysis/OE0532/21221/analysis/input/ribo_diff/sh40_vs_GFP_sh.metafile.csv
touch /icgc/dkfzlsdf/analysis/OE0532/21221/analysis/input/ribo_diff/CTN_sh_vs_GFP_sh.metafile.csv
```

Example of metafile: `vim $BASE_DIR/21221/analysis/input/ribo_diff/CTN_sh_vs_GFP_sh.metafile.csv`

```
Samples,Data_Type,Conditions
RP_GFP_sh_1,Ribo-Seq,Control
RP_GFP_sh_2,Ribo-Seq,Control
RP_GFP_sh_3,Ribo-Seq,Control
RP_CTN_sh_1,Ribo-Seq,Treated
RP_CTN_sh_2,Ribo-Seq,Treated
RP_CTN_sh_3,Ribo-Seq,Treated
RNA_GFP_sh_1,RNA-Seq,Control
RNA_GFP_sh_2,RNA-Seq,Control
RNA_GFP_sh_3,RNA-Seq,Control
RNA_CTN_sh_1,RNA-Seq,Treated
RNA_CTN_sh_2,RNA-Seq,Treated
RNA_CTN_sh_3,RNA-Seq,Treated
```

Copy metafiles to `21221_RNA` directory: 

```
for f in $(ls $BASE_DIR/21221/analysis/input/ribo_diff/*.csv); do echo "cp $f $BASE_DIR/21221_RNA/analysis/input/ribo_diff/"; done
```


## 3. Run RiboDiff 

```
source activate diricore
$BASE_DIR/software/ribo_diff/ribo_diff.sh 21221_RNA 21221 all_unique long
```

## 4. Cleanup

```
rm $BASE_DIR/21221_RNA/analysis/output/ribo_diff/te/all_unique/*.pkl
cp $BASE_DIR/21221_RNA/analysis/output/ribo_diff/te/all_unique/* $BASE_DIR/21221/analysis/output/ribo_diff/te/all_unique
cp $BASE_DIR/21221_RNA/analysis/output/ribo_diff/te/all_unique/*.pdf $BASE_DIR/21221_RNA/analysis/output/figures/ribo_diff
cp $BASE_DIR/21221/analysis/output/ribo_diff/te/*.pdf $BASE_DIR/21221/analysis/output/figures/ribo_diff

python $BASE_DIR/software/ribo_diff/fix_excel.py 21221_RNA all_unique
python $BASE_DIR/software/ribo_diff/fix_excel.py 21221 all_unique
```


## Highlight specific genes

1. create list of genes: `vim $BASE_DIR/21221_RNA/analysis/input/metadata/RPG_small.txt`

This will be the list:

```
RPSA
RPS2
RPS3
RPS3A
RPS4X
RPS4Y
RPS5
RPS6
RPS7
RPS8
RPS9
RPS10
RPS11
RPS12
RPS13
RPS14
RPS15
RPS15A
RPS16
RPS17
RPS18
RPS19
RPS20
RPS21
RPS23
RPS24
RPS25
RPS26
RPS27
RPS27A
RPS28
RPS29
RPS30
```

2. copy to RP: 
 
```
cp $BASE_DIR/21221_RNA/analysis/input/metadata/RPG_small.txt $BASE_DIR/21221/analysis/input/metadata/
```

3. highlight genes: 

```
module load R/3.6.2
module load gcc/7.2.0
Rscript $BASE_DIR/software/ribo_diff/highlight_genes.r 21221_RNA all_unique $BASE_DIR/21221_RNA/analysis/input/metadata/RPG_small.txt
```
