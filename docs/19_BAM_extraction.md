# BAM Extraction based on ID

For some analysis, BAM file format information of certain transcript or gene are necessary. To achieve this, the following script takes stable transcript/ gene ENSEMBL IDs as list input and performs list comparison.
The output is a BAM file of the selected transcripts/ genes.

USAGE: 
Locate your current working dir to the respective dir.

```
/omics/groups/OE0532/internal/Alex/scripts/BAMextract_with_geneList.sh /omics/groups/OE0532/internal/Alex/static/hg19/MT-transcripts.txt C159_toTranscriptome.bam
```


