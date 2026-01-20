# PRICE

This file aims to document the handling of PRICE (Probabilistic inference of codon activities by an EM algorithm) in the B250 department of DKFZ Heidelberg. 
Please check the original documentation of the Erhard Lab: https://github.com/erhard-lab/gedi/wiki/Price#preparing-a-reference

## 1. Re-Align fastq files
Input dir is: '$PROJECT_DIR/analysis/output/clean'
I edited this code to generate the file according to the new env…..basically, the loading of modules should be done before running this step

```
module load STAR/2.5.3a-GCC-14.1.0
module load SAMtools/1.20-GCC-14.1.0   

$BASE_DIR/scripts/scripts_andres/Align_to_transcriptome_PRICE_Debian.sh 44312_HCT hg19 80G

```

## 2. Index new bam files
```
for i in $(ls $BASE_DIR/3772/analysis/output/alignments/PRICE/toGenome/); do echo "bsub -q medium -R rusage[mem=30G] samtools index /omics/groups/OE0532/internal/Alex/3772/analysis/output/alignments/PRICE/toGenome/${i}"; done
```

## 3. Prepare dir architecture
```
mkdir -p /omics/groups/OE0532/internal/Andres/44312_HCT/analysis/output/PRICE/output/
cd /omics/groups/OE0532/internal/Andres/44312_HCT/analysis/output/PRICE/output

```

## 4. Run PRICE
```
module load R/4.4.3-GCCcore-14.1.0

```

```
for i in $(ls /omics/groups/OE0532/internal/Andres/44312_HCT/analysis/output/alignments/PRICE/toGenome/*.bam | xargs -n 1 basename); do echo "bsub -q long -R "rusage[mem=30G]" /omics/groups/OE0532/internal/Andres/scripts/scripts/Gedi/Gedi_1.0.5/gedi -e Price -D -reads /omics/groups/OE0532/internal/Andres/44312_HCT/analysis/output/alignments/PRICE/toGenome/${i} -genomic /omics/groups/OE0532/internal/Andres/scripts/scripts/Gedi/Gedi_1.0.5/hg19/hg19.oml -prefix ${i}/${i} -plot"; done
```
If needed, add '-progress' flag to receive a detailed list of the running progress.

Submit to cluster
```
bsub -q long -R rusage[mem=10G] /omics/groups/OE0532/internal/Alex//scripts/Gedi/Gedi_1.0.5/gedi -e Price -D -reads /omics/groups/OE0532/internal/Alex/3772/analysis/output/alignments/PRICE/toGenome/MCF7_Ctrl_toGenome.bam -genomic /omics/groups/OE0532/internal/Alex/scripts/Gedi/Gedi_1.0.5/hg19/hg19.oml -prefix MCF7_Ctrl_toGenome.bam/MCF7_Ctrl_toGenome.bam -progress -plot
```

PRICE will output various plots and files in the prefix-named dir. The most important file *${prefix}.orfs.tsv* contains all the information about all called ORFs.
Depending on desired output, interrogate the table with basic grep/cut/sort functions.

### 4.1 PRICE MM10: 

For mouse samples: Use mm10 reference file location for job submission.

```
bsub -q long -R rusage[mem=100G] /omics/groups/OE0532/internal/Alex/scripts/Gedi/Gedi_1.0.5/gedi -e Price -D -reads /omics/groups/OE0532/internal/Alex/39381/analysis/output/alignments/PRICE/toGenome/S1_E0771_MONO_toGenome.bam -genomic /omics/groups/OE0532/internal/Alex/genomes/PRICE_mm10/mm10.oml -prefix S1_E0771_MONO/S1_E0771_MONO -plot
```

### 4.2 PRICE Feature Quantification

To quantify all as significant predicted ORF features or Start Codons, create a new dir, copy the PRICE output tables into this directory and perform the respective counting script.
The script reads all files within this directory.
```
# eg.
mkdir -p $BASE_DIR/42445/analysis/output/PRICE/output/ORF_tables
cd $BASE_DIR/42445/analysis/output/PRICE/output/ORF_tables
cp ../S1_U2OS_Ctrl_toGenome.bam/S1_U2OS_Ctrl_toGenome.bam.orfs.tsv ./
/omics/groups/OE0532/internal/Alex//scripts/PRICE_Count_Categories.sh ./
# or
/omics/groups/OE0532/internal/Alex//scripts/PRICE_Count_Codons.sh ./
```

This will provide an output.tsv file in your current working directory about all ORF features/ start codons across all files in the input directory.

### 5 Transfer PRICE output to local folder

```
scp -r a690a@odcf-worker01.dkfz.de:/omics/groups/OE0532/internal/Andres/45659_AME/analysis/output/PRICE/* ~/analysis/45659_AME/

```


