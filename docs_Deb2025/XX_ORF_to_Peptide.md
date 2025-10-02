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

Run from the command line:

```
Rscript /omics/groups/OE0532/internal/Andres//scripts/scripts_andres/ORF_dna_to_peptide_table_v2.r <project_id> <species> <celltype1_celltype2_...> <min_peptide_length>
```


- `project_id`: Project folder name (e.g., `43979`)
- `species`: Genome build identifier (`mm10` or `hg19`)
- `celltype1_celltype2_...`: Underscore-delimited list of celltypes to analyze (e.g., `NP5_TC1`)
- `min_peptide_length`: Minimum peptide length to retain sequences and generate FASTA (e.g., `10`)

Example:

```
Rscript /omics/groups/OE0532/internal/Andres//scripts/scripts_andres/ORF_dna_to_peptide_table_v2.r 43979 mm10 NP5_TC1 10
```

---

## Analysis Pipeline Steps

1. **File loading:**  
   Reads all PRICE ORF output TSV files in the project folder excluding "ORF_tables" subfolders.

2. **Filtering:**  
   Keeps ORFs matching regex `"ORF"` in the `Type` column and with `p value < 0.05`.

3. **Chromosome, strand, and exon range parsing:**  
   Extracts genomic locations from the `Location` column, adjusting start positions by +1, honoring strand direction.

4. **Sequence extraction:**  
   Uses BSgenome objects (e.g., `BSgenome.Mmusculus.UCSC.mm10`) to extract exon sequences for each ORF, concatenating exons in correct order and applying reverse complement for negative strands.

5. **Translation:**  
   Spliced DNA sequences are translated to peptides using the `Biostrings::translate` function.

6. **Peptide and ORF properties:**  
   Calculates peptide length, stop codon position, and codon start concordance.

7. **Output per sample:**  
   Saves detailed TSV tables with DNA and peptide sequences for each sample separately. Generates peptide FASTA files for each sample applying minimum peptide length filtering.

8. **Summary plots:**  
   Produces four plots per sample: peptide length histogram, initiation codon frequencies, stop codon distribution, and ORF type frequencies; saved as PDFs.

9. **Merging and filtering by celltypes:**  
   Combines all per-sample data and filters by specified celltypes, consolidating treatments. Saves filtered combined tables per celltype.

10. **Unique peptide identification:**  
    For each celltype, identifies peptides unique to a single treatment group (based on peptide sequence) and writes a unique peptides table and corresponding FASTA file.

---

## Output Structure

All output files (tables, FASTA, plots) are saved under:

```
$BASE_DIR/<project_id>/analysis/output/PRICE/ORF_to_Peptide/
```

- `ORF_with_sequences_<SampleName>.tsv`  
- `peptides_<SampleName>.fasta`  
- `ORF_summary_plot_<SampleName>.pdf`  
- `ORF_with_sequences_<CellType>.tsv`  
- `unique_peptides_<CellType>.tsv`  
- `unique_peptides_<CellType>.fasta`  

---

## Notes

- The pipeline assumes correct installation of required Bioconductor and CRAN packages.
- Coordinate manipulation (+1) and strand-aware reverse complementation preserve genomic integrity.
- Filtering is customizable by altering `p value` and `min_peptide_length` parameters.
- Merging celltypes allows multi-treatment comparison while keeping sample-level resolution via individual outputs.

---

