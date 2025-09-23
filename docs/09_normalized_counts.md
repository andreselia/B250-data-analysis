# Normalized counts

> **_IMPORTANT:_** `$BASE_DIR` has to be specified in your `.bash_profile`. Details [here](docs/0_before_you_start.md)

## 0. Load samtools

This step requires [samtools](http://quinlanlab.org/tutorials/samtools/samtools.html). On the cluster just load samtools with the command:

```
module load samtools
```

## 1. Extract sequences from bam

```
$BASE_DIR/software/counts/1_get_seq_from_bam.sh 21221 all_unique
```

This will extract all reads from .bam and save them as .tsv file into this directory: `$BASE_DIR/21221/analysis/output/ext_diricore/all_unique/tsv/`

Parameters: `21221` - project_id, `all_unique` - bam_type (can be: `all` or `all_unique`). 

## 2. Normalize counts for each sample

```
python $BASE_DIR/software/counts/2_normalize_counts.py 21221 all_unique hg19
```

Parameters: `21221` - project_id, `all_unique` - bam_type, `hg19` - genome (to calculate RPKM we need to know the transcript length)

This script will calculate normalized counts (raw, CPM, TPM, RPKM) per gene and save this in a separate `.tsv` file for each sample.

Output dir: `$BASE_DIR//21221/analysis/output/alignments/reads_per_gene/normalized/all_unique/`

## 3. Aggregate samples into 1 file

```
python $BASE_DIR/software/counts/3_aggregate.py 21221 all_unique rpkm 50 
```

This will extract all genes which have at least 50 reads in each sample. 

The output will be written into this file: `/omics/groups/OE0532/internal/from_snapshot/21221/analysis/output/alignments/reads_per_gene/normalized//all_unique/rpkm_50min_reads.tsv`

## 4. Aggregate samples into 1 file and extract specific genes

Let's say we are only interested in a specific subset of genes, for example [this ones](http://ribosome.med.miyazaki-u.ac.jp/rpg.cgi?mode=orglist&org=Homo%20sapiens)

Let's create a file with the list of these genes: `vim $BASE_DIR/21221/analysis/input/metadata/RPG_small.txt`

These are the genes we are interested in:

```
RPSA
RPS2
RPS3
RPS3A
RPS4X
RPS4Y
RPS5
RPS6
RPS7
RPS8
RPS9
RPS10
RPS11
RPS12
RPS13
RPS14
RPS15
RPS15A
RPS16
RPS17
RPS18
RPS19
RPS20
RPS21
RPS23
RPS24
RPS25
RPS26
RPS27
RPS27A
RPS28
RPS29
RPS30
```

Now let's get the counts normalized to RPKM, only for this subset of genes:

```
python $BASE_DIR/software/counts/3_aggregate.py 21221 all_unique rpkm 0  $BASE_DIR/21221/analysis/input/metadata/RPG_small.txt gene_name
```

The parameters are the following: 

* `21221` - project_id

* `all_unique` - bam_type

* `rpkm` - type of normalization (can be: `rpkm`, `tpm`, `cpm` or `raw`)

* `0` - min number of reads. In this case we want to include all genes regardless the number of reads. 

* `$BASE_DIR/21221/analysis/input/metadata/RPG_small.txt` - path to the list of genes

* `gene_name` - how we want to represent the counts (possible options: `gene` or `gene_name`, `gene_id`, `trans` or `transcript` or `trans_id` or `transcript_id`)

Output will be written into this file: `/omics/groups/OE0532/internal/from_snapshot/21221/analysis/output/alignments/reads_per_gene/normalized//all_unique/RPG_small_rpkm_0min_reads.tsv`

And will look something like that: 

```
Gene	S11_CTN_sh_2	S12_CTN_sh_3	S13_BJ_hTert	S14_CoC_BJ_NC
RPS10	0.041584781034711686	0.0381723942119815	0.1316429135069198	0.12309305847171245
RPS11	0.27286380738491023	0.3306806854506312	1.0138292166187826	0.9918494501378088
RPS12	0.018189222766371516	0.0317442320358839	0.09703420248986903	0.0922905955608691
RPS13	0.1312908554419535	0.14388720556700113	0.5772403145144498	0.5062353367008943
RPS14	0.19238927703235692	0.2213198817112811	0.6930752683671597	0.6277710864887198
RPS15	0.567687256386192	0.6347806483248889	2.0679332702423925	1.5660305072818645
RPS15A	0.24750578066716217	0.3305755742523724	1.0124189491939368	1.0498423103986307
RPS16	0.25495281999110014	0.2579381609270348	0.8631743934593668	0.8311250784091808
RPS17	0.10509325061897798	0.07505944975437931	0.3730097281934005	0.34509139041358045
RPS18	0.13366993348215295	0.1590419453086988	0.4954919377664573	0.45981886101616765
RPS19	0.28466313611318533	0.31396701209933026	1.3199430556641318	1.1709345545726801
RPS2	0.08814251175105955	0.07836125677895234	0.4950999337210341	0.4531952636096438
RPS20	0.20072097898892624	0.31332945213276364	0.8820033865330359	0.7598633236764802
RPS21	0.2384493612221842	0.2488874038743994	0.42234063274307154	0.4175989710682177
RPS23	0.13283290429007893	0.10084966275357785	0.5951086614858655	0.5342631610278928
RPS24	0.2951671699931513	0.29508796362880874	0.8448193109063917	0.7842542774355924
RPS25	0.13188743804732828	0.1724324793477066	0.39307806789816807	0.35220709597984035
RPS26	0.0026738635959585215	0.013810686445233759	0.015988537020498156	0.008606353690026132
RPS27	0.08646307672737254	0.12288909558282116	0.17739948275785833	0.17037417780111103
```
