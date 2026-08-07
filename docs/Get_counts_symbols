# Get gene counts with gene symbols
This script translates ENST to Gene Symbols and generates a table with the read count for each gene

´´´
bsub -q medium -R "rusage[mem=30G]" $BASE_DIR/software/counts/get_counts_symbols.sh 47681 all_unique hg19

´´´
Output will look like this:

Processing sample: A549_BPTES
Processing sample: A549_CB839
Processing sample: A549_Ctrl
........
------------------------------------------------------------
Done. Output files:
  /omics/groups/OE0532/internal/....//47681/analysis/output/counts/all_unique/A549_BPTES.gene_counts.tsv
  /omics/groups/OE0532/internal/....//47681/analysis/output/counts/all_unique/A549_CB839.gene_counts.tsv
.........
Transcript-to-gene map used:
  /omics/groups/OE0532/internal/....//47681/analysis/output/counts/all_unique/transcript_to_gene.hg19.tsv
