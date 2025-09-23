# Cellphone DB

This analysis is based on the tool [Cellphone DB](https://github.com/Teichlab/cellphonedb) 

The installation instructions are [here](https://github.com/Teichlab/cellphonedb)

## 1. Extract the gene counts from .bam

```
/icgc/dkfzlsdf/analysis/OE0532/software/ext_diricore/1_get_seq_from_bam.sh 17246 all_unique
```

This script takes .bam files from `/icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/alignments/toTranscriptome/` and extracts the reads. 

The output files will be written in this directory: `/icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/ext_diricore/all_unique/tsv`

Each file contains 3 columns: transcript id, coordinate of the 5' end of each read, and the sequence. Example:

```
ENSMUST00000192183      98      AATGGATCTATGTACACCACG
ENSMUST00000192183      340     TCATAGCCATCCAAACCTTCTGGTACCAG
ENSMUST00000192183      766     TCGGACACTTCAAGTTCTGCATCACTCTCTG
ENSMUST00000192183      768     GGACACTTCAAGTTCTGCATCACTCTCTG
ENSMUST00000192183      775     TCAAGTTCTGCATCACTCTCTG
ENSMUST00000192692      2082    CGGGCCTTCGCTATGCTTTGTTTTA
ENSMUST00000195384      26      CAACCGGATCTGCAAGGTGCTGGCTGTGAACCAG
ENSMUST00000195384      27      AACCGGATCTGCAAGGTGCTGGCTG
```

## 2. Get counts per transcript and normalize

```
python /icgc/dkfzlsdf/analysis/OE0532/software/scripts/normalize_counts1.py 17246 all_unique hg19
```

The script will take the reads from the previous step, from the directory `/icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/ext_diricore/all_unique/tsv`

It will group reads by transcript, normalize to CPM, RPKM and TPM, and write the output to this directory: `/icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/alignments/reads_per_gene/normalized/all_unique`

Example of output: 

```
transcript      gene_name       gene_id raw_counts  cpm     tpm     rpkm
ENST00000000233.5       ARF5    ENSG00000004059.6       205     2.2452119759387752e-11  31.046694561926593      0.020355502955020626
ENST00000000412.3       M6PR    ENSG00000003056.3       440     4.818991558112493e-11   26.669230423721572      0.0174854555809597
ENST00000000442.6       ESRRA   ENSG00000173153.9       25      2.738063385291189e-12   1.8853991136819377      0.0012361459978741261
ENST00000001008.4       FKBP4   ENSG00000004478.5       189     2.069975919280139e-11   13.92667063001743       0.009130903922717861
ENST00000001146.2       CYP26B1 ENSG00000003137.4       7       7.66657747881533e-13    0.24711000217783974     0.00016201558492847276
ENST00000002125.4       NDUFAF7 ENSG00000003509.11      34      3.723766203996017e-12   2.6100993980034324      0.0017112896158069935
ENST00000002165.6       FUCA2   ENSG00000001036.9       547     5.990882687017122e-11   38.78368409393215       0.025428194766626156
ENST00000002501.6       DBNDD1  ENSG00000003249.9       2       2.1904507082329513e-13  0.16069876043503575     0.0001053607844267894
```

## 3. Aggregate all samples into 1 file 

```
python /icgc/dkfzlsdf/analysis/OE0532/software/scripts/normalize_counts2.py 17246 all_unique rpkm
```

This will create a file: `/icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/alignments/reads_per_gene/normalized/all_unique/rpkm.tsv`

Example of output:

```
Gene    MRC5_G100       MRC5_G10        159_G10 159_G100
ENSG00000000003 0.004071100591004126    0.004423024601247635    0.006653824872131573    0.0070721194242481045
ENSG00000000419 0.08294415754342041     0.07403078435124584     0.08047291314313094     0.09760658330762795
ENSG00000000457 0.002489698124451966    0.0025817742350913763   0.0016248856683883691   0.003924889108108417
ENSG00000000460 0.007303567828340383    0.005626573028758278    0.007220805280969843    0.009615099508365343
ENSG00000000971 0.05079946083794536     0.052098454096356925    0.03467419563882983     0.046970733095848066
ENSG00000001036 0.04605020243229234     0.04903388863819462     0.045758354805998595    0.046532306553569505
ENSG00000001084 0.013538407382483976    0.013863778232591063    0.018300293311855214    0.024287318599945838
ENSG00000001167 0.006518400518724701    0.006611903183681494    0.009478815954796802    0.008825296502200522
```

## 4. Create meta file 
`vim  /icgc/dkfzlsdf/analysis/OE0532/17246/analysis/input/metadata/cellphonedb_meta.txt`

```
Cell    cell_type
MRC5_G100       MRC5_G100
MRC5_G10        MRC5_G10
159_G10 159_G10
159_G100        159_G100
```

## 5. Run cellphone db 

```
module load R/3.6.2 && 
    cellphonedb method statistical_analysis /icgc/dkfzlsdf/analysis/OE0532/17246/analysis/input/metadata/cellphonedb_meta.txt 
    /icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/alignments/reads_per_gene/normalized/all_unique/rpkm.tsv 
    --output-path /icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/figures/cellphonedb
```

This will output the results into this directory: `/icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/figures/cellphonedb`

Those files are expected to be there:

```
deconvoluted.txt
means.txt
pvalues.txt
significant_means.txt
```


## 6. Dotplot 

```
cellphonedb plot dot_plot --means-path /icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/figures/cellphonedb/means.txt 
                          --pvalues-path /icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/figures/cellphonedb/pvalues.txt 
                          --output-path /icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/figures/cellphonedb 
                          --columns /icgc/dkfzlsdf/analysis/OE0532/17246/analysis/input/metadata/cellphonedb_columns.txt
```

This will create the file: `/icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/figures/cellphonedb/dotplot.pdf`

![](/pics/cellphonedb_1.png)

## 7. Heatmap 

```
cellphonedb plot heatmap_plot 
            --pvalues-path /icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/figures/cellphonedb/pvalues.txt 
            --output-path /icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/figures/cellphonedb 
            /icgc/dkfzlsdf/analysis/OE0532/17246/analysis/input/metadata/cellphonedb_meta.txt
```

This will create the following files:

```
/icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/figures/cellphonedb/heatmap_count.pdf
/icgc/dkfzlsdf/analysis/OE0532/17246/analysis/output/figures/cellphonedb/heatmap_log_count.pdf
```

![](/pics/cellphonedb_2.png)

![](/pics/cellphonedb_3.png)
