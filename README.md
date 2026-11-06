# long-read-variant-calling

[![Nextflow](https://img.shields.io/badge/nextflow-%E2%89%A525.04.0-23aa62.svg)](https://www.nextflow.io/)
[![Run with singularity](https://img.shields.io/badge/run%20with-singularity-1d355c.svg?labelColor=000000)](https://sylabs.io/docs/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24022536.svg)](https://doi.org/10.5281/zenodo.24022536)

A Nextflow DSL2 pipeline for small-variant and structural-variant (SV) calling from long-read sequencing data (Oxford Nanopore and PacBio HiFi), with a single MultiQC report summarising alignment and variant QC across samples.

The pipeline is developed for non-human research genomes. *Saccharomyces cerevisiae* (R64 / S288C reference) is the model organism it is built and tested around, but any organism with a reference FASTA can be used.

## Pipeline overview

```
FASTQ (one file per sample)
  │
  ▼
MINIMAP2 ── alignment, sort, index, samtools stats
  │
  ├─▶ PEPPER-Margin-DeepVariant ── SNVs/indels ──▶ bcftools stats, RTG vcfstats
  ├─▶ Clair3 ───────────────────── SNVs/indels ──▶ bcftools stats, RTG vcfstats
  ├─▶ cuteSV ───────────────────── SVs ──────────▶ RTG vcfstats, SV type counts, Samplot preview
  └─▶ Sniffles2 ────────────────── SVs ──────────▶ RTG vcfstats, SV type counts, Samplot preview
  │
  ▼
MULTIQC ── one HTML report for all samples, including software versions
```

Each caller can be switched on or off independently (`--run_pepper`, `--run_clair3`, `--run_cutesv`, `--run_sniffles2`).

## Quick start

### Requirements

- [Nextflow](https://www.nextflow.io/) ≥ 25.04 (tested with 25.10.2)
- [Singularity/Apptainer](https://apptainer.org/)

> **Note:** a pure `-profile conda` run is not supported yet. Most processes only define a container, and Samplot and the SV summary step only define a conda environment. See the [issue tracker](https://github.com/gencorefacility/long-read-variant-calling/issues).

### Prepare the reference

The reference FASTA must have a samtools index (`.fai`) next to it:

```bash
samtools faidx genome.fa     # creates genome.fa.fai
```

### Run

```bash
git clone https://github.com/gencorefacility/long-read-variant-calling.git

nextflow run long-read-variant-calling/main.nf \
    --input     'reads/*.fastq.gz' \
    --reference genome.fa \
    --outdir    results \
    --minimap2_preset map-ont \
    -profile singularity,slurm \
    -resume
```

Quote the `--input` glob so Nextflow expands it rather than the shell. Each FASTQ file is treated as one sample, and the sample ID is taken from the file name.

## Parameters

### Required

| Parameter | Description |
|---|---|
| `--input` | FASTQ file(s), glob allowed. One file = one sample. |
| `--reference` | Reference genome FASTA. A `.fai` index must exist alongside it. |
| `--outdir` | Output directory. |
| `--minimap2_preset` | Minimap2 preset: `map-ont`, `lr:hq` (recent ONT, ~Q20+), or `map-hifi`. |

### Callers

| Parameter | Default | Description |
|---|---|---|
| `--run_pepper` | `true` | PEPPER-Margin-DeepVariant small variants. |
| `--run_clair3` | `false` | Clair3 small variants. |
| `--run_cutesv` | `true` | cuteSV structural variants. |
| `--run_sniffles2` | `true` | Sniffles2 structural variants. |

### Caller options

| Parameter | Default | Description |
|---|---|---|
| `--pepper_preset` | `--ont_r10_q20` | PEPPER data-type preset (e.g. `--ont_r9_guppy5_sup`, `--hifi`). |
| `--clair3_model` | – | Path to a Clair3 model directory matching your basecaller/chemistry. Required with `--run_clair3`. |
| `--clair3_platform` | `ont` | `ont` or `hifi`. |
| `--tandem_repeats` | – | Optional tandem-repeat BED for Sniffles2; improves SV calls in repetitive regions. |
| `--samplot_max_plots` | `5` | Number of SVs per caller rendered with Samplot (see the note under Outputs). |
| `--email` | – | Address to email a run summary to on completion. |

Tool-specific arguments (minimum support, minimum SV size, MAPQ filters, resources) are set per process in `nextflow.config` via `ext.args`, `cpus`, `memory` and `time`.

## Profiles

| Profile | Description |
|---|---|
| `singularity` | Run each process in its Singularity/Apptainer container. |
| `slurm` | Submit jobs to SLURM. Combine with a container profile, e.g. `-profile singularity,slurm`. |
| `standard` | Run locally (default). |

## Outputs

Results are written to `--outdir`, one directory per process:

```
results/
├── minimap2/                        # <sample>.bam, .bam.bai, .stats
├── pepper/                          # <sample>.pepper.vcf.gz (+ .tbi)
├── clair3/                          # <sample>.clair3.vcf.gz (+ .tbi)
├── cutesv/                          # <sample>.cutesv.vcf
├── sniffles2/                       # <sample>.sniffles2.vcf.gz, <sample>.snf
├── bcftools_stats_<caller>/
├── rtg_vcfstats_<caller>/
├── sv_length_distribution_<caller>/ # per-type SV counts and length summaries
├── samplot_<caller>/                # SV preview images
├── multiqc/
│   └── multiqc_report.html          # main report
└── reports/                         # Nextflow execution report, timeline, trace, DAG
```

**About the Samplot images:** these are a *preview* of the first few SVs in each VCF, not a curated selection. In yeast, the first SVs on a chromosome usually fall in repetitive subtelomeric regions, so these images are rarely the most informative ones. Use a genome browser for proper SV review.

## Known limitations

Tracked on the [issue tracker](https://github.com/gencorefacility/long-read-variant-calling/issues). The main ones:

- No samplesheet yet; one FASTQ per sample, and runs of the same sample must be concatenated beforehand.
- The reference `.fai` is not generated automatically.
- Genotyping assumes diploid samples; there is no ploidy option yet.
- No CNV / aneuploidy detection yet.
- `-profile conda` is not fully supported.

## Citation

If you use this pipeline, please cite it:

> Khalfan M. *long-read-variant-calling*: a Nextflow pipeline for long-read small-variant and structural-variant calling. Zenodo. https://doi.org/10.5281/zenodo.24022536

This DOI always resolves to the latest release. To cite a specific version, use the version DOI listed on the [Zenodo record](https://doi.org/10.5281/zenodo.24022536). GitHub's "Cite this repository" button gives the same citation in APA and BibTeX.

Please also cite the tools it runs:

- **Nextflow**: Di Tommaso P, et al. (2017) Nextflow enables reproducible computational workflows. *Nat Biotechnol* 35:316–319.
- **minimap2**: Li H. (2018) Minimap2: pairwise alignment for nucleotide sequences. *Bioinformatics* 34:3094–3100.
- **SAMtools / BCFtools**: Danecek P, et al. (2021) Twelve years of SAMtools and BCFtools. *GigaScience* 10:giab008.
- **PEPPER-Margin-DeepVariant**: Shafin K, et al. (2021) Haplotype-aware variant calling with PEPPER-Margin-DeepVariant enables high accuracy in nanopore long-reads. *Nat Methods* 18:1322–1332.
- **Clair3**: Zheng Z, et al. (2022) Symphonizing pileup and full-alignment for deep learning-based long-read variant calling. *Nat Comput Sci* 2:797–803.
- **cuteSV**: Jiang T, et al. (2020) Long-read-based human genomic structural variation detection with cuteSV. *Genome Biol* 21:189.
- **Sniffles2**: Smolka M, et al. (2024) Detection of mosaic and population-level structural variants with Sniffles2. *Nat Biotechnol* 42:1571–1580.
- **RTG Tools**: Cleary JG, et al. (2015) Comparing variant call files for performance benchmarking of next-generation sequencing variant calling pipelines. *bioRxiv* 023754.
- **Samplot**: Belyeu JR, et al. (2021) Samplot: a platform for structural variant visual validation and automated filtering. *Genome Biol* 22:161.
- **MultiQC**: Ewels P, et al. (2016) MultiQC: summarize analysis results for multiple tools and samples in a single report. *Bioinformatics* 32:3047–3048.

## Contributing

Bug reports and feature requests go in [Issues](https://github.com/gencorefacility/long-read-variant-calling/issues). To contribute code:

1. Fork the repository and keep your fork's `main` in sync with upstream `main`.
2. Create one branch per issue from an up-to-date `main` (e.g. `fix/reference-channel`).
3. Open a pull request against `gencorefacility/long-read-variant-calling:main` and reference the issue it closes (`Fixes #12`).

## Authors and acknowledgments

- **Author:** Mohammed Khalfan ([@mohammedkhalfan](https://github.com/mohammedkhalfan)), NYU Center for Genomics and Systems Biology (CGSB) Genomics Core
- **Maintainer:** Eric Borenstein ([@genericdata](https://github.com/genericdata)), NYU High Performance Computing and Department of Biology

Thanks to the Nextflow and nf-core communities, whose conventions this pipeline follows.

## License

MIT. See [LICENSE](LICENSE).
