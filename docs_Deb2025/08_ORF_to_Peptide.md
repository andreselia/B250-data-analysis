# PRICE ORF Peptide Analysis Script

## Overview

This R script automates the analysis of open reading frames (ORFs) detected by the PRICE pipeline, extracting and translating genomic sequences into peptides. It processes multiple sample files from a project, generates DNA and peptide sequences per ORF, filters and annotates them, and produces summary statistics and visualizations. Additionally, it provides peptide FASTA files per sample and identifies peptides unique to specific treatments within celltypes.

---

## Purpose

- Extract precise spliced DNA sequences for each ORF based on exon ranges with strand specificity and coordinate adjustments.
- Translate DNA sequences into peptide sequences using standard genetic code.
- Filter ORFs based on statistical significance and peptide length.
- Generate peptide FASTA files for each sample.
- Summarize peptide and ORF properties visually with customizable plots.
- Create merged tables filtered by celltype, consolidating treatments.
- Identify unique peptides per celltype present in only one treatment group and export them with corresponding FASTA files.

---

## Requirements

- R (version >= 4.0 recommended)
- Bioconductor packages: `GenomicRanges`, `Biostrings`, `BSgenome.Hsapiens.UCSC.hg19`, `BSgenome.Mmusculus.UCSC.mm10`
- CRAN packages: `readr`, `dplyr`, `stringr`, `ggplot2`, `patchwork`

Ensure that the environment variable `BASE_DIR` is set pointing to the base folder containing your project data structure.

---

## Usage

```

module load R/4.4.3-GCCcore-14.1.0

```

Run from the command line:

```
 bsub -q long -R rusage[mem=40G] Rscript /omics/groups/OE0532/internal/Andres//scripts/scripts_andres/ORF_dna_to_peptide_table_v4.r <project_id> <species> <celltype1_celltype2_...> <min_peptide_length>
```


- `project_id`: Project folder name (e.g., `43979`)
- `species`: Genome build identifier (`mm10` or `hg19`)
- `celltype1_celltype2_...`: Underscore-delimited list of celltypes to analyze (e.g., `NP5_TC1`)
- `min_peptide_length`: Minimum peptide length to retain sequences and generate FASTA (e.g., `10`)

Example:

```
 bsub -q long -R rusage[mem=40G] Rscript /omics/groups/OE0532/internal/Andres//scripts/scripts_andres/ORF_dna_to_peptide_table_v4.r 43979 mm10 NP5_TC1 10
```

---

## Analysis Pipeline Steps

## Step 1 — Process each PRICE file (in parallel)
 
For every `*.orfs.tsv` file found:
 
1. Reads PRICE's columns (`Gene`, `Id`, `Location`, `Candidate Location`, `Codon`, `Type`, `Start`, `Range`, `p value`, `Total`).
2. Assigns a unique `Sample` name based on the file's full relative path (prevents two files that share a parent-folder name from overwriting each other).
3. Parses `Location` to extract chromosome, strand, and the coordinates of each exon (an ORF can have several exons separated by `|`). PRICE's start coordinates are 0-based, so the script adds **+1** to each exon's start position to convert it to the 1-based coordinates that `getSeq()`/`GRanges` expect. The end coordinate is used as-is.
4. **Extracts the spliced DNA sequence** (`Spliced_DNA`) from the reference genome, joining exons together and reverse-complementing when the strand is `-`.
5. **Translates to peptide** (`Peptide_Sequence`), discarding sequences with invalid characters.
6. Computes `Peptide_Length`.
**Output of this step:**
- `full_<sample>.tsv` — one table per input file, with all original PRICE columns plus `Spliced_DNA`, `Peptide_Sequence`, `Peptide_Length`, `chr`, `strand`, `ranges`.
## Step 2 — Combine by celltype
 
For each celltype (e.g. `BJ`, `Sen`):
 
1. Merges all `full_<sample>.tsv` files belonging to that celltype.
2. Extracts `CellType` and `Treatment` from the sample name (a sample with no treatment suffix — e.g. `Sen` on its own — gets `Treatment = "none"`).
**Output:**
- `full_combined_<celltype>.tsv` — all samples of that celltype together, unfiltered.
## Step 3 — Filter and split by treatment
 
On the combined table, filters:
- Excludes `Type` values `Variant`, `ncRNA`, `orphan`, `Trunc`, `CDS` (keeps the non-canonical ORFs: uORF, uoORF, iORF, etc.)
- `p value < 0.05`
- `Peptide_Length >= min_peptide_length` (the script's 4th argument)
For each treatment within the celltype:
 
**Output (per celltype + treatment):**
- `filtered_<celltype>_<treatment>.tsv` — filtered rows for that combination
- `filtered_peptides_<celltype>_<treatment>.fasta` — peptides in FASTA format (header `>Sample_Id`)
- `plots_<celltype>_<treatment>.pdf` — 4 plots: peptide length distribution by Type, combined length distribution, Type frequency (%), start-codon frequency
- `table_peptide_length_by_type_<celltype>_<treatment>.tsv` — frequency table behind the first plot
## Step 4 — Unique vs. shared peptides
 
On the already-filtered table (Step 3), groups by peptide sequence:
 
- **Unique** (`n_distinct(Treatment) == 1`): peptides found in only one treatment.
- **Shared** (`n_distinct(Treatment) > 1`): peptides present in more than one treatment of the same celltype.
**Output (per celltype):**
- `unique_peptides_<celltype>.tsv` / `.fasta`
- `shared_peptides_<celltype>.tsv` / `.fasta` (header includes the IDs and treatments where each shared peptide appears)
---
 
## Summary of generated files
 
| File | Level | Content |
|---|---|---|
| `full_<sample>.tsv` | per sample | full table with DNA + translated peptide |
| `full_combined_<celltype>.tsv` | per celltype | all samples merged, unfiltered |
| `filtered_<celltype>_<treatment>.tsv` | celltype+treatment | rows passing the significance/type/length filters |
| `filtered_peptides_<celltype>_<treatment>.fasta` | celltype+treatment | filtered peptides in FASTA |
| `plots_<celltype>_<treatment>.pdf` | celltype+treatment | exploratory plots |
| `table_peptide_length_by_type_<celltype>_<treatment>.tsv` | celltype+treatment | length-by-Type frequency table |
| `unique_peptides_<celltype>.tsv` / `.fasta` | celltype | peptides exclusive to one treatment |
| `shared_peptides_<celltype>.tsv` / `.fasta` | celltype | peptides present in 2+ treatments |
 
## Notes / known limitations
 
- **Coordinate convention:** PRICE reports exon start positions 0-based; the script converts them to 1-based (`start + 1`) before querying the genome. If you ever compare raw `Location` values against the extracted `Spliced_DNA`/genome coordinates by hand, remember the start you see in `Location` is 1 less than what was actually fetched.
- If a celltype or celltype+treatment combination has no rows left after filtering, that file simply isn't generated (a message is logged, it's not an error).
- The script assumes a `<CellType>_<Treatment>` naming convention in PRICE's file/folder names; a sample with no `_` is treated as `Treatment = "none"`.
- Requires that duplicate/stale run folders (e.g. an old `ZT/` run) already be removed from the input directory beforehand — the script doesn't auto-detect or exclude them, it only excludes `ORF_tables/` and `ORF_Files/`.
