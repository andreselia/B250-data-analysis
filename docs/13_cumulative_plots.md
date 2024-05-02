# Cumulative plots

## 1. Get sequences from bam

```
$BASE_DIR/software/ext_diricore/1_get_seq_from_bam.sh 22276 all_unique
```

## 2. Get cumulative plot

```
bsub -q long -R "rusage[mem=20G]" python $BASE_DIR/software/ext_diricore/cumulative/33_cumulative_reads.py 22276 all_unique $BASE_DIR/22276/analysis/input/metadata/MT-genes.txt
```

This script will plot each gene from the list in a separate pdf file. Additionally, it will output the table with the counts per position.

The list can include either gene names, gene IDs, or transcript IDs.

In our case the file ` $BASE_DIR/22276/analysis/input/metadata/MT-genes.txt` looks as following:

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
For mm10 samples: 

```
bsub -q long -R "rusage[mem=20G]" python $BASE_DIR/scripts/cumulative_reads_mm10.py 39381 all_unique /omics/groups/OE0532/internal/Alex/ID_lists/Mito_mm10_geneSymbols.txt
```
The cumulative script uses a transcript length file specific for mouse or human, as well as a mouse-specific gene list.


## 3. Get cumulative plot for a subset of samples

```
bsub -q long -R "rusage[mem=20G]" python $BASE_DIR/software/ext_diricore/cumulative/33_cumulative_reads.py 22276 all_unique $BASE_DIR/22276/analysis/input/metadata/MT-genes.txt pool_24
```

Here we specify the pool name, which is `pool_24`. To make the script run properly, it is required to have the file `$BASE_DIR/22276/analysis/input/metadata/rpf_density_samplenames_pool_24.tsv`

Only samples specified in the file, will be plotted. 

Example: 

![](/pics/cumulative_plot.png)
