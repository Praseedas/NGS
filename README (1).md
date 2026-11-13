# NGS Genomic Data Processing Pipeline

An end-to-end short-read NGS workflow, run in Google Colab, that takes paired-end FASTQ files for a single proband sample through quality control, trimming, alignment, and variant calling and filtering against the hg19 chromosome 8 reference.

**Notebook:** `NGS_Genomic_Data_Processing_Pipeline.ipynb`

---

## Overview

| Item | Detail |
|---|---|
| Sample | `proband` (single sample) |
| Input | Paired-end FASTQ: `proband_R1.fq.gz`, `proband_R2.fq.gz` |
| Reference | hg19 (GRCh37) chromosome 8, `hg19_chr8.fa.gz` |
| Data source | Zenodo record 3243160 |
| Platform | Google Colab (Linux, 2 threads used) |
| Final output | Filtered VCF: `variants/filtered/proband_filtered.vcf` |

## Workflow

```
Paired-end FASTQ
      │
Raw QC (FastQC + MultiQC)
      │
Read trimming / filtering (fastp)
      │
Post-trim QC (FastQC + MultiQC)
      │
Alignment (BWA-MEM)
      │
SAM → BAM → sort → index (Samtools)
      │
Alignment QC (flagstat, coverage, stats)
      │
Variant calling (FreeBayes)
      │
Variant filtering (BCFtools: QUAL ≥ 30, DP ≥ 10)
      │
Filtered VCF + summary stats + figures
```

## Tools and versions

Versions as recorded in the notebook's execution log:

| Tool | Version | Purpose |
|---|---|---|
| FastQC | 0.12.1 | Read quality assessment |
| MultiQC | pip install (version not recorded) | Aggregated QC reports |
| fastp | 0.23.4 | Trimming and quality filtering |
| BWA | 0.7.17-r1188 | Indexing and BWA-MEM alignment |
| Samtools | 1.19.2 | BAM conversion, sorting, indexing, QC |
| BCFtools | 1.19 | Variant filtering and statistics |
| FreeBayes | 1.3.10 | Variant calling (installed via micromamba, bioconda) |

## Project structure

The notebook creates this layout under `/content/NGS_Pipeline`:

```
NGS_Pipeline/
├── data/
│   ├── raw/            # downloaded FASTQ files
│   └── processed/      # fastp-trimmed FASTQ files
├── reference/          # hg19 chr8 FASTA + BWA index
├── qc/
│   ├── raw/            # FastQC + MultiQC on raw reads
│   └── trimmed/        # fastp report + FastQC + MultiQC on trimmed reads
├── alignment/          # SAM, BAM, sorted BAM + index, flagstat/coverage/stats
├── variants/
│   ├── raw/            # FreeBayes output (proband_raw.vcf)
│   └── filtered/       # filtered VCF + bcftools stats
├── logs/               # execution_log.txt
├── scripts/
└── reports/            # Figures 1–4 (PNG, 300 dpi)
```

## Pipeline steps and parameters

1. **Setup:** create the directory tree, install FastQC, fastp, BWA, Samtools and BCFtools via `apt`, and install FreeBayes via micromamba (`conda-forge` + `bioconda`).
2. **Data download:** `wget` the two FASTQ files and the chr8 reference from Zenodo. Gzip integrity of all three files is checked (`gzip -t`).
3. **Raw QC:** `fastqc -t 2`, aggregated with `multiqc`.
4. **Trimming:** `fastp --detect_adapter_for_pe --qualified_quality_phred 20 --length_required 50 --thread 2`
5. **Post-trim QC:** FastQC and MultiQC repeated on the trimmed reads.
6. **Alignment:** `bwa index` on the reference, then `bwa mem -t 2` with read group `@RG\tID:proband\tSM:proband\tPL:ILLUMINA`.
7. **BAM processing:** `samtools view` (SAM → BAM), `sort`, `index`.
8. **Alignment QC:** `samtools flagstat`, `coverage`, `stats`.
9. **Variant calling:** `freebayes -f hg19_chr8.fa proband_sorted.bam` with default settings.
10. **Variant filtering:** `bcftools view -i 'QUAL>=30 && INFO/DP>=10'`
11. **Reporting:** `bcftools stats`, an execution log, and four summary figures.

## Results

### Read QC and trimming (fastp)

| Metric | Value |
|---|---|
| Reads per mate before filtering | 2,392,554 |
| Reads per mate after filtering | 2,147,877 |
| Reads passed filter (R1 + R2) | 4,295,754 |
| Failed: low quality | 487,556 |
| Failed: too many N | 428 |
| Failed: too short | 1,370 |
| Reads with adapter trimmed | 5,206 |
| Q30 bases, R1 (before → after) | 90.23% → 95.22% |
| Q30 bases, R2 (before → after) | 86.81% → 93.50% |
| Duplication rate | 20.79% |

### Alignment (Samtools)

| Metric | Value |
|---|---|
| Total reads | 4,296,659 |
| Mapped reads | 4,295,952 (99.98%) |
| Properly paired | 4,087,090 (95.14%) |
| Singletons | 707 (0.02%) |
| Mean depth (chr8) | 2.96X |
| Chromosome 8 bases covered | 3.55% |
| Mean base quality / mean MAPQ | 35.8 / 53.6 |

### Variants (FreeBayes → BCFtools)

| Metric | Count |
|---|---|
| Raw variant records | 15,550 |
| Filtered variant records (QUAL ≥ 30, DP ≥ 10) | 4,789 |
| SNPs | 4,109 |
| MNPs | 90 |
| Indels | 576 |
| Other | 37 |
| Multiallelic sites | 65 |

## Figures

Saved to `reports/`:

- `Figure1_Pipeline_Workflow.png`: workflow diagram
- `Figure2_Alignment_QC.png`: mapped, properly paired, and chromosome coverage percentages
- `Figure3_Variant_Distribution.png`: filtered variants by type
- `Figure4_Variant_Filtering.png`: raw vs filtered variant counts

## How to run

1. Open the notebook in Google Colab.
2. Run the cells in order. The setup cells install all tools; no local installation is needed.
3. Mount Google Drive when prompted if you want the final backup/ZIP steps to work. The notebook writes to `MyDrive/Genomic_Data_Analyst_Assignment/`; change `DRIVE_DIR` and the backup paths if you want a different location.

**Resources:** the full project folder reached about 3.2 GB (the uncompressed `proband.sam` alone is about 1.4 GB), so a standard Colab runtime with roughly 12 GB RAM and 100 GB disk is sufficient.

## Limitations and notes

- **Low coverage.** Mean depth is about 3X and only about 3.6% of chr8 is covered, so variant calls are low-confidence and many genotypes will be unreliable. Results should be treated as candidate calls, not confirmed variants.
- **Single sample, single chromosome.** There is no trio or cohort context, and the analysis is restricted to chr8.
- **No duplicate marking.** fastp reports a 20.79% duplication rate, but no duplicate-marking step (e.g., `samtools markdup`) is run before variant calling.
- **Basic filtering only.** Filtering uses QUAL and DP thresholds alone; there is no strand-bias, allele-balance, or population-frequency filtering and no annotation step.
- **Notebook state.** Some cells (tool installs, MultiQC, reference decompression, version checks) appear more than once because of re-runs during development. Run order matters: `bwa index` must follow reference decompression and BWA installation.
- **Software versions** in the execution log are typed into the log cell rather than captured programmatically, so confirm them against the cell outputs if exact provenance matters.
