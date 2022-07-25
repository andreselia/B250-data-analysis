# RiboDiff **_NEW_** Pseudo-Replicates/ dTEG

> **_IMPORTANT:_** `$BASE_DIR` has to be specified in your `.bash_profile`. Details [here](docs/0_before_you_start.md)

> **_IMPORTANT 2:_** [RiboDiff](https://github.com/kate-v-stepanova/RiboDiff) has to be installed!

## 1. Obtain and Process RNAseq data 
## 2. Pseudo-Replicates
## 3. Preparing data input for RiboDiff / dTEG
## 4. Run RiboDiff /dTEG





## 1. Obtain and Process RNAseq data 

Locate and identify the important fastq files and symlink to internal processing directory:

```
ln -s /some/input/dir/xyz.fastq $BASE_DIR/analysis/input/fastq/new_name.fastq
```

Run cutadapt to reduce adapter sequences in your RNAseq dataset: e.g.

```
bsub -q medium -R "rusage[mem=80G]" cutadapt -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -m 30 -o $BASE_DIR/CellSignaling_RNA/analysis/output/trimmed/Ctu2KO_DMSO_R1.fastq -p $BASE_DIR/CellSignaling_RNA/analysis/output/trimmed/Ctu2KO_DMSO_R2.fastq --cores=6 $BASE_DIR/CellSignaling_RNA/analysis/input/fastq/AS-X-R1.fastq.gz $BASE_DIR/CellSignaling_RNA/analysis/input/fastq/AS-X-R2.fastq.gz
```
Flags:
-o: Output Read 1
-p: Output Read 2

Last section:
Read 1 and 2 input (AS-X-R1.fastq.gz and AS-X-R2.fastq.gz)




## 2. Pseudo-Replicates

This analysis requires replicates. If replicates cannot be provided, .tsv files obtained from "get_seq_from_bam.py" can used as input for the following script. The script shuffles data lines to new files.
Input: `"$project_path/analysis/output/ext_diricore/$bam_type/tsv/Split/input/"`
Output: `"$project_path/analysis/output/ext_diricore/$bam_type/tsv/Split/output/"` 

```
bsub -q medium -R "rusage[mem=30G]" /omics/groups/OE0532/internal/Alex/scripts/PseudoReplicate_Split.sh CellSignaling_RP all_unique

```

