# Read size distribution

> **_IMPORTANT:_** `$BASE_DIR` has to be specified in your `.bash_profile`. Details [here](docs/0_before_you_start.md)


## 1. Extract sequences from bam

Skip this step if this was already done.

```
bsub -q long -R "rusage[mem=50G]" $BASE_DIR/software/counts/1_get_seq_from_bam.sh 21221 all_unique
```

The script will extract reads from the .bam files. `21221` - dataset_id, `all_unique` - bam_type (`all` - all reads including duplicates, `all_unique` - only unique reeads)

Output dir: `BASE_DIR/21221/analysis/output/ext_diricore/all_unique/tsv/`

## 2. Get size distribution

```
bsub -q long  -R "rusage[mem=50G]" python $BASE_DIR/software/counts/4_size_distribution.py 21221 all_unique hg19
```

`21221` - dataset_id

`all_unique` - bam_type

`hg19` - genome version (default: hg19)

Output dir: `$BASE_DIR/21221/analysis/output/figures/size_dist/all_unique/all_genes/`


## 3. Get size distribution for a subset of genes

Let's say, we want to see the size distribution only for Mitochondrial genes.

Create a list of genes: `vim $BASE_DIR/21221/analysis/input/metadata/MT-genes.txt`

```
MT-ND1
MT-ND2
MT-CO1
MT-CO2
MT-ATP8
MT-ATP6
MT-CO3
MT-ND3
MT-ND4L
MT-ND4
MT-ND5
MT-ND6
MT-CYB
```

This list can contain either gene names (e.g. `MT-ND1`), gene ids (`ENSG00000198888.2`), or transcript ids (`ENST00000361390.2`)

Get size distribution for MT-genes: 

```
python $BASE_DIR/software/counts/4_size_distribution.py 21221 all_unique hg19 $BASE_DIR/21221/analysis/input/metadata/MT-genes.txt
```

Output dir: `$BASE_DIR/21221/analysis/output/figures/size_dist/all_unique/MT-genes/`

Example of the plot: 

![](/pics/size_distribution.png)

## 4. Get size distribution excluding a given subset of genes

Let's say, we want to get size distribution only for cytosolic genes (all genes excluding mitochondrial genes).

We have a list of mitocchondrial genes here: `$BASE_DIR/21221/analysis/input/metadata/MT-genes.txt`

Let's get the size distribution:

```
python $BASE_DIR/software/counts/4_size_distribution.py 21221 all_unique hg19 $BASE_DIR/21221/analysis/input/metadata/MT-genes.txt exclude
```
