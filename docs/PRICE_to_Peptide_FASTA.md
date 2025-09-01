# ORF and Peptide Analysis Script

## Overview

This R script processes PRICE output data to identify open reading frames (ORFs) and generate peptide sequences based on DNA extracted from reference genomes. It expands ORF genomic ranges, filters by significance, extracts DNA sequences for ORFs, translates them to peptides, saves peptide and ORF details per cell type, and generates summary plots and peptide FASTA files.

---

## Usage

Run the script via command line on an HPC or local terminal with:

```

bsub -q long -R rusage[mem=40G] Rscript /omics/groups/OE0532/internal/Andres/scripts/scripts_andres/ORF_dna_to_peptide_table.r <project_id> <species> <celltype1_celltype2_...>

```

- Replace `<project_id>` with your project's numeric or string ID.
- Replace `<species>` with the genome build name: either `mm10` (mouse) or `hg19` (human).
- Replace `<celltype1_celltype2_...>` with underscore-separated cell types to subset, e.g., `NP5_TC1`.

---

## Input Description

- **Environment Variable**: `BASE_DIR` must be defined and points to the top-level directory containing project data.
- **PRICE Output Files**: The script expects `.tsv` files from PRICE located under:
  
```

${BASE_DIR}/${project_id}/analysis/output/PRICE/output/

```

It searches recursively for files ending with `_toGenome.bam.orfs.tsv` excluding any in `ORF_tables` subfolders.

---

## Script Workflow

### 1. Reading and Filtering ORFs

- Reads all PRICE ORF files in the project directory.
- Selects relevant columns: Gene, Id, Location, Candidate Location, Codon, Type, Start, Range, p value, and Total.
- Filters ORFs by:
- **Type containing the string "ORF"** (e.g., uORF, dORF, iORF) to retain only open reading frames.
- **p-value less than 0.05** to keep statistically significant ORFs.

This filtering ensures that only significant and relevant ORFs proceed for downstream analysis.

### 2. Expanding ORF Coordinates

- Splits multi-range ORFs into individual genomic ranges.
- Adjusts start coordinates by +1 for `+` strand ORFs.
- Produces expanded ORFs with explicit coordinates to facilitate sequence extraction.

### 3. Extracting DNA Sequences

- Uses `GenomicRanges` to create genomic ranges.
- Filters ranges to chromosomes supported by the selected genome (`mm10` or `hg19`).
- Extracts DNA sequences for the ORFs using the appropriate `BSgenome` package.

### 4. Translating to Peptides

- Converts DNA sequences to peptide sequences with codon-aware translation.
- Calculates peptide length and flags peptides containing stop codons.

### 5. Saving Per Cell Type

- Extracts cell type and treatment information from sample names by splitting on underscore.
- Saves peptide and ORF tables per requested cell type as TSV files.
- Generates peptide FASTA files named by cell type.

### 6. Generating Summary Plots

- Creates for each sample:
  - Overlay histogram of peptide length by strand.
  - Bar plots of initiation codon frequency by strand.
  - Bar plots of stop codon presence by strand.
  - Bar plots of ORF type frequency.
- Saves plots as PDF files in the output folder.

---

## Outputs

- TSV files per cell type: `ORF_DNA_to_Peptide_<CellType>.tsv`
- Peptide FASTA files per cell type: `unique_peptides_<CellType>.fasta`
- Per-sample summary plots: `plot_<SampleName>.pdf`

All output files are saved to:

```

${BASE_DIR}/${project_id}/analysis/output/PRICE/output/

```

---

## Dependencies

- R packages: `readr`, `dplyr`, `stringr`, `GenomicRanges`, `Biostrings`, `ggplot2`, `patchwork`
- Genome packages: `BSgenome.Mmusculus.UCSC.mm10`, `BSgenome.Hsapiens.UCSC.hg19`

Make sure these are installed before running the script.

---

## Notes

- The script requires an environment variable `BASE_DIR` to be set in the shell or session.
- The filtering in step 1 (reading ORFs) is critical to retain only ORFs relevant for peptide generation: those containing `"ORF"` in the Type field and having a p-value less than 0.05.
- The cell types argument should list all cell types of interest, separated by underscores.
- The script dynamically handles both mouse and human genomes based on the `species` argument.

---

## Contact

For support or questions, please contact the script author or project lead.

```
