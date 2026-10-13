# PRICE joint analysis

`run_price_joint.sh` runs [PRICE](https://github.com/erhard-lab/gedi/wiki/Price) on **all Ribo-seq BAM files of a project in a single job**, using a `.bamlist` file. The result is one ORF table with the same ORF candidates for every sample and one read-count column per sample, ready for downstream comparison between conditions.

## Why a joint run?

Running PRICE separately on each BAM gives every sample its own set of ORF candidates, its own codon model and its own significance test. Tables from separate runs cannot be merged reliably: an ORF missing from one table may have zero reads, or may simply never have been tested.

A joint run instead produces:

- one shared set of ORF candidates across all samples
- one read-count column per sample, plus a `Total` column
- one p value per ORF computed from all samples together, so ORFs translated in only one condition can still be detected

## Input and output folders

The script derives all paths from the project ID:

```
${BASE_DIR}/<PROJID>/analysis/output/alignments/PRICE/toGenome/*.bam   # input
${BASE_DIR}/<PROJID>/analysis/output/PRICE/joint_analysis/             # output
```

Every `.bam` file in the input folder is included in the run. The BAM files must be sorted, indexed and mapped to the genome selected with `SPECIES`.

## Usage

The script is shared, so there is nothing to install or copy. Run it directly from its location:

```bash
$BASE_DIR/scripts/scripts_andres/run_price_joint.sh <PROJID> <SPECIES>
```

| Argument  | Description                              | Example    |
| --------- | ---------------------------------------- | ---------- |
| `PROJID`  | Project ID; also used as the folder name | `48138_AE` |
| `SPECIES` | Reference genome: `hg19` or `mm10`        | `hg19`     |

The script checks its inputs, writes the BAM list and submits a single LSF job. When the job finishes, LSF sends the job report and log to your usual email address.

### Example

```bash
$ $BASE_DIR/scripts/scripts_andres/run_price_joint.sh 48138_AE hg19
Project:  48138_AE
Species:  hg19  (/omics/groups/OE0532/internal/Andres/scripts/scripts/Gedi/Gedi_1.0.5/hg19/hg19.oml)
Output:   /omics/groups/OE0532/internal/Andres/48138_AE/analysis/output/PRICE/joint_analysis/48138_AE_hg19_joint.*
Samples included (6), in output column order:
.../toGenome/HT29_ctrl2_toGenome.bam
.../toGenome/HT29_ctrl3_toGenome.bam
.../toGenome/HT29_ctrl4_toGenome.bam
.../toGenome/HT29_iri2_toGenome.bam
.../toGenome/HT29_iri3_toGenome.bam
.../toGenome/HT29_iri4_toGenome.bam
Job <54364131> is submitted to queue <long>.
```

### Optional settings

Resources and extra PRICE flags are set through environment variables placed before the command. Nothing in the script needs to be edited.

| Variable     | Default | Description                                              |
| ------------ | ------- | -------------------------------------------------------- |
| `THREADS`    | `8`     | Cores reserved in LSF and passed to PRICE (`-nthreads`)  |
| `LSF_MEM`    | `30G`   | Memory reserved in LSF                                   |
| `QUEUE`      | `long`  | LSF queue                                                |
| `EXTRA_OPTS` | (empty) | Additional PRICE flags, e.g. `-percond`                  |

Example:

```bash
THREADS=16 LSF_MEM=60G EXTRA_OPTS="-percond" \
  $BASE_DIR/scripts/scripts_andres/run_price_joint.sh 48138_AE hg19
```

The resource summary in the LSF email reports the maximum memory the job used. Use it to adjust `LSF_MEM` for future runs.

`-percond` fits a separate codon model for each sample instead of one shared model. Consider it when conditions or libraries differ in quality or in read-length distribution.

### Supported species

| `SPECIES` | PRICE reference                                                  |
| --------- | ---------------------------------------------------------------- |
| `hg19`    | `$BASE_DIR/scripts/scripts/Gedi/Gedi_1.0.5/hg19/hg19.oml`         |
| `mm10`    | `/omics/groups/OE0532/internal/Alex/genomes/PRICE_mm10/mm10.oml` |

To add a species, add one line to the `case` block in the script:

```bash
hg38) GENOME=/path/to/PRICE_hg38/hg38.oml ;;
```

> **Important:** the species must match the genome the BAM files were mapped to. PRICE does not check this; a mismatch produces meaningless results without an error.

## What the script does

1. Selects the PRICE reference from `SPECIES` and the input and output folders from `PROJID`.
2. Checks that `BASE_DIR` is defined and that the `gedi` launcher, the reference and the BAM folder exist.
3. Writes all `*.bam` files of the project to `<RUN_NAME>.bamlist`, one per line.
4. Stops with an error if any BAM has no index.
5. Prints a summary and the sample list. The order of this list is the column order in the output table.
6. Loads the R module (needed for `-plot`) and submits one LSF job that runs:

```bash
gedi -e Price -D \
  -reads <RUN_NAME>.bamlist \
  -genomic <reference>.oml \
  -prefix <OUT_DIR>/<RUN_NAME> \
  -nthreads ${THREADS} ${EXTRA_OPTS} -plot
```

`RUN_NAME` is `<PROJID>_<SPECIES>_joint`, for example `48138_AE_hg19_joint`. It is also the LSF job name, so it appears in the subject of the email you receive when the job finishes.

## Output

All files are written to `${BASE_DIR}/<PROJID>/analysis/output/PRICE/joint_analysis/`:

| File                                | Content                                                      |
| ----------------------------------- | ------------------------------------------------------------ |
| `<RUN_NAME>.bamlist`                | Input BAM files, in output column order                      |
| `<RUN_NAME>.orfs.tsv`               | **Main result:** all ORF candidates with per-sample counts   |
| `<RUN_NAME>.orfs.cit`               | Called ORFs in CIT format (viewer, conversion to BED)        |
| `<RUN_NAME>.orfs.cit.metadata.json` | Metadata: condition names, total read numbers, etc.          |
| `<RUN_NAME>.codons.cit`             | Inferred codon activities                                     |
| `<RUN_NAME>.*.rmq`                  | Index files for the genome browser                            |
| `<RUN_NAME>.codons.bin`, `.orfs.bin`, `.start.model` | Intermediate files                          |

### Columns of `.orfs.tsv`

| Column               | Meaning                                                                                  |
| -------------------- | ---------------------------------------------------------------------------------------- |
| `Gene`               | Gene ID (overlapping genes are joined with `_`)                                          |
| `Id`                 | `<transcript>_<type>_<number>`; the transcript in which the ORF was found                 |
| `Location`           | ORF coordinates from start to stop codon (0-based, end exclusive; introns separated by `\|`) |
| `Candidate Location` | ORF candidate before start-codon prediction                                              |
| `Codon`              | Start codon                                                                              |
| `Type`               | ORF category: `CDS`, `Ext`, `Trunc`, `Variant`, `uORF`, `uoORF`, `iORF`, `dORF`, `ncRNA`, `orphan` |
| `Start`, `Range`     | Start-codon prediction scores (higher is better)                                         |
| `p value`            | Generalized binomial test, **not** corrected for multiple testing                        |
| one column per BAM   | Estimated read counts for that sample                                                    |
| `Total`              | Sum over all samples                                                                     |

## Notes for downstream analysis

- **Counts are fractional.** PRICE assigns reads to codons probabilistically (EM algorithm) and weights multimapping reads, so values such as `42.2` are expected. Round them before using count-based tools such as DESeq2.
- **Correct the p values** (e.g. Benjamini-Hochberg) and keep only significant ORFs. `.orfs.tsv` contains all candidates, including those without evidence of translation.

## References

- PRICE documentation: <https://github.com/erhard-lab/gedi/wiki/Price>
- Erhard F, *et al.* Improved Ribo-seq enables identification of cryptic translation events. *Nature Methods* 15, 363–366 (2018).
