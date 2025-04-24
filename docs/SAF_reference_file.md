# SAF Reference File

Feature counting from custom genomic coordinates can be achieved with featureCounts and an input SAF file.
To obtain SAF format, perform the following:

## 1. Perform PRICE and CoordinateConversion.py
Please refer to ORF Reference and PRICE in this Git documentation for further information

## 2. Formatting
After the step of header removal subsequent to the CoordinateConversion.py script, re-format for SAF:

```
awk 'BEGIN {OFS="\t"; print "GeneID", "Chr", "Start", "End", "Strand"} {OFS="\t"; print $6, $1, $2, $3, $5}' input.bed > output.saf
```

This SAF file can be used for counting:

```
bsub -q long -R "rusage[mem=35G]" featureCounts -a output.saf  -F SAF -o FeatureCounts_output.tsv input1_toGenome.bam input2_toGenome.bam inputx_toGenome.bam
```

### Important consideration:
If these counts are supposed to be used for cross-sample comparison approaches, it is recommend to merge the output.saf with a SAF file covering CDS genomic regions to sufficiently cover comparable total read counts per sample. 
Total read counts are utilized by eg. DEseq to calculate normalization factors. Conditional total read count differences in the custom genomic region can bias this normalization. Thus, combination of the custom SAF with a CDS SAF will reduce this bias.
