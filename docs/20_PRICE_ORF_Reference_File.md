# ORF Reference File

This section provides input about how to obtain an ORF reference file from the PRICE output.
Please refer to the PRICE section in prior to obtain ORF prediction data. Needed output is:

```
*.orfs.tsv
```

This process was manually setup step-by-step in 2022 and 2023. Future streamlining of all steps into one single script is highly recommended. 

## 1. ORF feature extraction
Next, extract important information about significant features of interest, eg. uORFs. For this, use the awk command below and modify as needed:

```
awk -F'\t' '{OFS="\t"; if (NR == 1) {print "Gene", "Id", "Location", "Candidate Location", "Codon", "Type", "Start", "Range", "p value", "File", "Total";} else if ($6 == "uORF" && $9 < 0.05) {print $0; }}' input.orfs.tsv > Significant_features_output.orfs.tsv
```

## 2. Genomic Coordinates Format Conversion and Merging
The following steps convert the PRICE output files to bed, bed6, bed12, nucleotide and peptide FASTA formats. Merging of peptide FASTA files will results in the creation of a final database. 
Next, copy or symlink the *Significant_features_output.orfs.tsv* files to a newly created directory, eg.:

```
mkdir -p $BASE_DIR/FASTA_references/Sen_dB/conversion_input/
cd $BASE_DIR/FASTA_references/Sen_dB/
```

The following script will read the specified input tsv file in the *conversion_input* directory and perform a format conversion, with *Sen_dB* beeing the sub-directory in *FASTA_references*. 
The script will output files in the bed annotation format.

```
bsub -q medium -R rusage[mem=2G] python /omics/groups/OE0532/internal/Alex//scripts/CoordinateFormatConversion.py Sen_dB Sign_uoORF_A549_Alis_40325_toGenome.bam.orfs.tsv
```

Once all files were processed, remove the header line from the bed output, because this would create issues in the downstream process.

```
for i in $(ls ./*bed*); do echo "sed -i 1d ${i}"; done
```

Next, bed files are converted to bed6 and bed12:

```
awk 'BEGIN{FS=OFS="\t"} {print $1, $2, $3, $4, 0, $5} input.bed > output.bed6
/omics/groups/OE0532/internal/Alex//scripts/bed6Tobed12.sh -i output.bed6 > output.bed12
```

## 3. Nucleotide and Peptide FASTA

Next, bed12 files are converted to nucleotide FASTA files using *bedtools getfasta*. Important, define the path to your genome annotation FASTA file (here: ~/genomes/hg19/GRCh37.p13.genome.fa)

```
module load bedtools
module load R/4.2.0
module load gcc/7.2.0
bsub -q long -R "rusage[mem=10G]" bedtools getfasta -fi ~/genomes/hg19/GRCh37.p13.genome.fa -bed ./input.bed12 -s -name -split -fo ./output.fasta
```

If needed, FASTA files can be merged now, since merging before *CoordinateFormatConversion.py* will lead to indexing errors:
```
cat file1.fasta file2.fasta file3.fasta fileX.fasta > Merged_nucleotide.fasta
```

Next, convert nucleotide FASTA to peptide FASTA
```
mkdir -p $BASE_DIR/FASTA_references/Sen_dB/DNA
mkdir -p $BASE_DIR/FASTA_references/Sen_dB/Peptide
mv ./*.fasta $BASE_DIR/FASTA_references/Sen_dB/DNA
/omics/groups/OE0532/internal/Alex//scripts/faTrans $BASE_DIR/FASTA_references/Sen_dB/DNA/Merged_nucleotide.fasta $BASE_DIR/FASTA_references/Sen_dB/Peptide/Merged_peptide_non_unique.fasta
```

Filter all non-unique peptide sequences. 
```
/omics/groups/OE0532/internal/Alex/scripts/FastaToTbl.sh $BASE_DIR/FASTA_references/Sen_dB/Peptide/Merged_peptide_non_unique.fasta | sort -k2 -u | /omics/groups/OE0532/internal/Alex/scripts/TblToFasta.sh > $BASE_DIR/FASTA_references/Sen_dB//Peptide/Merged_peptide_unique.fasta
```

Using the *cat* command from above, files can be merged to one file. The Tbl conversion introduces new line breaks in the fasta file. Thus, the last step will be merging peptide sequence lines together, which will result in the correct FASTA format of one line identifier and one line sequence information.

```
/omics/groups/OE0532/internal/Alex//scripts/Merge_Fasta_peptide_lines.sh $BASE_DIR/FASTA_references/Sen_dB/Merged_peptide_unique.fasta > $BASE_DIR/FASTA_references/Sen_dB/Final_Merged_peptide_unique.fasta
```





