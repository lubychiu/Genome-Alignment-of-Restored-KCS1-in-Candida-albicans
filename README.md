# Genome-Alignment-of-Restored-KCS1-in-Candida-albicans
This repository contains a reproducible, end-to-end bioinformatics pipeline designed to verify the genomic architecture of restored KCS1 Candida albicans strains.
In this genetic rescue experiment, the native KCS1 gene locus on Chromosome R was knocked out. To restore strain functionality, a functional copy of the KCS1 expression cassette was integrated into a known neutral locus located on chromosome 5. 

The workflow is divided into five steps executed on a high-performance computing (HPC) cluster running the SLURM workload manager:

1. Quality Control (FastQC): Evaluates raw paired-end sequence data for adapter contamination and quality decay.
2. Adapter & Quality Trimming (Trimmomatic): Strips Illumina adapters and filters low-quality bases using parallel SLURM Array Jobs.
3. Reference Genome Indexing & Mapping (Bowtie2): Builds a comprehensive genomic index from the NCBI Candida albicans reference and aligns the trimmed reads.
4. Coordinate Sorting & Targeted Extraction (SAMtools): Converts heavy SAM alignments into binary BAM format, sorts them by genomic coordinates, and extracts clean, subsetted BAM files isolated specifically to chromosome R and chromosome 5.
5. Visual Verification (IGV): Loads the compact, localized alignment tracks onto a local desktop browser to verify the native knockout boundaries and confirm spanning reads across the new integration junctions.

Using next-generation sequencing (NGS) data from three separate candidate strains ("Restore6", "Restore7", and "Restore8"), this pipeline automates read quality control, adapter trimming, whole-genome alignment, and targeted locus extraction. The primary objective is to visually confirm seamless integration and proper coverage at the insertion boundaries using a genome browser.
