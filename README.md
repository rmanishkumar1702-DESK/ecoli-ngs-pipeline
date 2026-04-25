# E. coli K12 NGS Variant Calling and Annotation Pipeline

## Overview
An end-to-end Next Generation Sequencing (NGS) variant calling and annotation pipeline built from scratch using publicly available data from NCBI SRA. This project covers the complete bioinformatics workflow from raw sequencing reads to fully annotated variants with biological interpretation.

---

## Organism and Data
| Parameter | Details |
|-----------|---------|
| Organism | *Escherichia coli* K12 MG1655 |
| Reference Genome | NC_000913.3 (GCF_000005845.2_ASM584v2) |
| Genome Size | 4,641,652 bp |
| SRA Accession | SRR2584863 |
| Data Source | NCBI SRA (Sequence Read Archive) |
| Sequencing Type | Whole Genome Sequencing (WGS) |

---

## Tools and Versions
| Tool | Version | Purpose |
|------|---------|---------|
| SRA Toolkit | 3.0.3 | Download raw sequencing data from NCBI SRA |
| FastQC | 0.12.1 | Quality control of raw reads |
| fastp | 1.1.0 | Read trimming and quality filtering |
| BWA | 0.7.13 | Read alignment to reference genome |
| SAMtools | 1.21 | BAM file processing — sort, index, flagstat |
| bcftools | 1.23.1 | Variant calling and filtering |
| SnpEff | 4.3t | Variant annotation |
| IGV | - | Alignment and variant visualization |

---

## Pipeline Workflow

```
Raw SRA Data (NCBI)
        ↓
   SRA Toolkit (prefetch + fastq-dump)
        ↓
   FastQC (Quality Control)
        ↓
   fastp (Read Trimming)
        ↓
   BWA (Alignment → SAM file)
        ↓
   SAMtools (SAM → BAM → Sort → Index)
        ↓
   bcftools mpileup + call (Variant Calling → BCF/VCF)
        ↓
   bcftools filter (Variant Filtering → filtered.vcf)
        ↓
   SnpEff (Annotation → annotated.vcf)
        ↓
   IGV (Visualization)
```

---

## Step by Step Commands

### Step 1 — Download SRA Data
```bash
prefetch SRR2584863
fastq-dump --split-files SRR2584863
```

### Step 2 — Quality Control
```bash
fastqc SRR2584863_1.fastq SRR2584863_2.fastq
```

### Step 3 — Read Trimming
```bash
fastp -i SRR2584863_1.fastq -I SRR2584863_2.fastq \
      -o SRR2584863_1_trimmed.fastq -O SRR2584863_2_trimmed.fastq \
      -h SRR2584863_fastp.html -j SRR2584863_fastp.json
```

### Step 4 — Index Reference Genome
```bash
bwa index ref_genome.fna
```

### Step 5 — Alignment
```bash
bwa mem ref_genome.fna SRR2584863_1_trimmed.fastq SRR2584863_2_trimmed.fastq > alignment.sam
```

### Step 6 — BAM Processing
```bash
samtools view -bS alignment.sam > alignment.bam
samtools sort alignment.bam -o alignment_sorted.bam
samtools index alignment_sorted.bam
samtools flagstat alignment_sorted.bam > flagstat.txt
```

### Step 7 — Variant Calling
```bash
bcftools mpileup -f ref_genome.fna alignment_sorted.bam -o raw.bcf
bcftools call -mv -o variants.vcf raw.bcf
```

### Step 8 — Variant Filtering
```bash
bcftools filter -s PASS variants.vcf -o filtered.vcf
```

### Step 9 — Build Custom SnpEff Database
```bash
# Create database directory
mkdir -p ~/miniconda3/envs/ngs_analysis/share/snpeff-4.3.1t-1/data/NC_000913.3

# Copy files
cp ref_genome.fna ~/miniconda3/envs/ngs_analysis/share/snpeff-4.3.1t-1/data/NC_000913.3/sequences.fa
cp GCF_000005845.2_ASM584v2_genomic.gff ~/miniconda3/envs/ngs_analysis/share/snpeff-4.3.1t-1/data/NC_000913.3/genes.gff

# Add to snpEff.config
echo "NC_000913.3.genome : Escherichia_coli_k12" >> snpEff.config
echo "NC_000913.3.chromosomes : NC_000913.3" >> snpEff.config

# Build database
snpEff build -gff3 -v NC_000913.3
```

### Step 10 — Variant Annotation
```bash
snpEff ann -stats snpEff_summary.html NC_000913.3 filtered.vcf > annotated.vcf
```

### Step 11 — Verify Annotation Quality
```bash
grep "ERROR_CHROMOSOME_NOT_FOUND" annotated.vcf | wc -l
# Expected output: 0
```

---

## Key Results

### Summary Statistics
| Parameter | Value |
|-----------|-------|
| Total Variants | 34,565 |
| Variant Rate | 1 per 134 bases |
| Total Effects | 381,681 |
| Annotation Errors | 0 ✅ |
| Ts/Tv Ratio | 2.5651 ✅ |

### Variants by Type
| Type | Count | Percentage |
|------|-------|-----------|
| SNP | 34,298 | 99.2% |
| Insertion | 130 | 0.4% |
| Deletion | 137 | 0.4% |

### Effects by Impact
| Impact | Count | Percentage |
|--------|-------|-----------|
| HIGH | 83 | 0.022% |
| MODERATE | 5,090 | 1.334% |
| LOW | 24,857 | 6.513% |
| MODIFIER | 351,651 | 92.132% |

### Functional Class
| Class | Count | Percentage |
|-------|-------|-----------|
| Silent | 24,653 | 82.9% |
| Missense | 5,083 | 16.9% |
| Nonsense | 37 | 0.1% |
| Missense/Silent Ratio | 0.2045 | Normal ✅ |

### Top HIGH Impact Genes
| Gene | HIGH Impact | Effect Type | Biological Note |
|------|------------|-------------|----------------|
| yfbL | 3 | Frameshift | Most impacted — predicted inner membrane protein |
| yzcX | 2 | Stop gained | Hypothetical protein |
| yhaV | 2 | Stop gained | Hypothetical protein |
| yriA | 2 | Start lost + Frameshift | Double hit — completely destroyed |
| ydF | 2 | Frameshift | Heavily mutated gene |

---

## Key Biological Findings

1. **SNP dominant dataset** — 99.2% SNPs consistent with bacterial whole genome resequencing
2. **Normal mutation pattern** — Missense/Silent ratio of 0.2045 indicates natural genomic variation with no strong selection pressure
3. **High quality variants** — Ts/Tv ratio of 2.5651 confirms real biological mutations (not sequencing artifacts)
4. **yfbL is most impacted gene** — 3 frameshift mutations suggest this predicted inner membrane protein is likely non-functional in this strain
5. **All HIGH impact genes are 'y' prefix genes** — hypothetical/uncharacterized proteins in E. coli K12 suggesting these are non-essential genes where mutations accumulate freely
6. **yriA double hit** — both start_lost and frameshift = most severely damaged gene in dataset
7. **554 heterozygous variants** — suggests possible mixed bacterial subpopulations in sample

---

## Notable Technical Challenge

A key challenge in this project was the **chromosome not found error** in SnpEff annotation. No pre-built SnpEff database matched the NC_000913.3 reference genome used in this study.

**Solution:** Built a custom SnpEff database from scratch using:
- Reference genome FASTA (NC_000913.3)
- Gene annotation GFF file (GCF_000005845.2_ASM584v2)
- Manual configuration of snpEff.config

This resolved all 34,565 chromosome errors resulting in **0 annotation errors**.

---

## Environment
- OS: Windows 11 with WSL (Ubuntu)
- Conda Environment 1: trimming_env — fastp, BWA, SAMtools, SRA Toolkit
- Conda Environment 2: ngs_analysis — bcftools, SnpEff, IGV
- FastQC — installed separately on Windows (D drive)
---

## Acknowledgements
- Raw sequencing data obtained from NCBI SRA: SRR2584863
- Reference genome obtained from NCBI GenBank: NC_000913.3 (GCF_000005845.2_ASM584v2) — E. coli K12 MG1655
- Gene annotation GFF file obtained from NCBI RefSeq: GCF_000005845.2_ASM584v2_genomic.gff
- All data used is publicly available through NCBI (National Center for Biotechnology Information)

---

## Author
**Manishkumar R**
M.Sc. Biotechnology
Chennai, Tamil Nadu, India
📧 rmanishkumar1702@gmail.com

