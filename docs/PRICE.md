# PRICE

This file aims to document the handling of PRICE (Probabilistic inference of codon activities by an EM algorithm) in the B250 department of DKFZ Heidelberg. 
Please check the original documentation of the Erhard Lab: https://github.com/erhard-lab/gedi/wiki/Price#preparing-a-reference

## 1. Re-Align fastq files
Input dir is: '$PROJECT_DIR/analysis/output/clean'
```
$BASE_DIR/scripts/Align_to_transcriptome_PRICE.sh 3808 hg19 80G
```

## 2. Index new bam files
```
for i in $(ls $BASE_DIR/3772/analysis/output/alignments/PRICE/toGenome/); do echo "bsub -q medium -R rusage[mem=30G] samtools index /omics/groups/OE0532/internal/Alex/3772/analysis/output/alignments/PRICE/toGenome/${i}"; done
```

## 3. Prepare dir architecture
```
mkdir -p /omics/groups/OE0532/internal/Alex/3772/analysis/output/PRICE/output/
cd /omics/groups/OE0532/internal/Alex/3772/analysis/output/PRICE/output
```

## 4. Run PRICE
```
module load R/4.2.0
module load java
```

```
for i in $(ls /omics/groups/OE0532/internal/Alex/3772/analysis/output/alignments/PRICE/toGenome/); do echo "bsub -q long -R "rusage[mem=30G]" /omics/groups/OE0532/internal/Alex/scripts/Gedi/Gedi_1.0.5/gedi -e Price -D -reads /omics/groups/OE0532/internal/Alex/3772/analysis/output/alignments/PRICE/toGenome/${i} -genomic hg19 -prefix ${i}/${i} -plot"; done
```
If needed, add '-progress' flag to receive a detailed list of the running progress.

Submit to cluster
```
bsub -q long -R rusage[mem=10G] /omics/groups/OE0532/internal/Alex//scripts/Gedi/Gedi_1.0.5/gedi -e Price -D -reads /omics/groups/OE0532/internal/Alex/3772/analysis/output/alignments/PRICE/toGenome/MCF7_Ctrl_toGenome.bam -genomic hg19 -prefix MCF7_Ctrl_toGenome.bam/MCF7_Ctrl_toGenome.bam -progress -plot
```

PRICE will output various plots and files in the prefix-named dir. The most important file *${prefix}.orfs.tsv* contains all the information about all called ORFs.
Depending on desired output, interrogate the table with basic grep/cut/sort functions.


