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

## Paths
- Raw FASTQ files: /home/yc1201/genome_alignment_kcs1/data/raw
- Reference FASTA: /home/yc1201/genome_alignment_kcs1/data/reference
- FastQC result output: /home/yc1201/genome_alignment_kcs1/results/qc
- SAM/BAM files: /home/yc1201/genome_alignment_kcs1/results/aligned
- Scripts: /home/yc1201/genome_alignment_kcs1/slurm

## Acquire FASTQ files of 3 candidates and upload to HPC
- Kcs1Restore6_S235_R1_001.fastq.gz, Kcs1Restore6_S235_R2_001.fastq.gz
- Kcs1Restore7_S236_R1_001.fastq.gz, Kcs1Restore7_S236_R2_001.fastq.gz
- Kcs1Restore8_S237_R1_001.fastq.gz, Kcs1Restore8_S237_R2_001.fastq.gz

## Quality control (FASTQC)
```bash
# Enter interactive mode on a compute node (from where you are)
$ srun --pty bash

# Make output directory
$ mkdir -p /home/yc1201/genome_alignment_kcs1/results/qc

# Load fastqc
$ module load fastqc

# Confirm fastqc is available:
$ fastqc -h

# Run FastQC on all uncompressed fastq files in data/raw/
$ fastqc -o /home/yc1201/genome_alignment_kcs1/results/qc/ /home/yc1201/genome_alignment_kcs1/data/raw/*.fastq.gz

$ echo "Quality control complete."
```
```bash
# Download FastQC reports for visualization
# Create local directory for reports
$ mkdir -p ~/Downloads/kcs1_fastqc_reports

#Download HTML files from the HPC
$ gcloud compute scp m12-controller:/home/yc1201/genome_alignment_kcs1/results/qc/*.html ~/Downloads/kcs1_fastqc_reports
```

## Trimmomatic
Use Trimmomatic to clean up raw files using results from FastQC.
```bash
# Write slurm script
$ nano sbatch.trimmomatic
```
In the window, write slurm script to submit job.
```bash
#!/bin/bash
#SBATCH --job-name=trimmomatic_kcs1
#SBATCH --output=/home/yc1201/genome_alignment_kcs1/slurm/%x.o%j
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=yc1201@georgetown.edu
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --time=04:00:00
#SBATCH --mem=8G
#SBATCH --array=6-8

# Establish base directory paths
$ BASE_DIR="/home/yc1201/genome_alignment_kcs1"
$ RAW_DIR="${BASE_DIR}/data/raw"
$ OUT_DIR="${BASE_DIR}/results/trimmed"

# Create output folder if it doesn't exist
$ mkdir -p ${OUT_DIR}

# Map the SLURM Array ID (6, 7, 8) to your specific sample numbers
$ if [ ${SLURM_ARRAY_TASK_ID} -eq 6 ]; then
    SAMPLE="Kcs1Restore6_S235"
$ elif [ ${SLURM_ARRAY_TASK_ID} -eq 7 ]; then
    SAMPLE="Kcs1Restore7_S236"
$ elif [ ${SLURM_ARRAY_TASK_ID} -eq 8 ]; then
    SAMPLE="Kcs1Restore8_S237"
$ fi

# Load Trimmomatic module ("aliases" needed for GU HPC setup here) 
$ shopt -s expand_aliases 
$ module load trimmomatic

# Run Trimmomatic PE using absolute paths
$ trimmomatic PE -threads 2 \
$ "${RAW_DIR}/${SAMPLE}_R1_001.fastq.gz" \
$ "${RAW_DIR}/${SAMPLE}_R2_001.fastq.gz" \
$ "${OUT_DIR}/${SAMPLE}_R1_paired.fq.gz" "${OUT_DIR}/${SAMPLE}_R1_unpaired.fq.gz" \
$ "${OUT_DIR}/${SAMPLE}_R2_paired.fq.gz" "${OUT_DIR}/${SAMPLE}_R2_unpaired.fq.gz" \
$ LEADING:10 \
$ TRAILING:10 \
$ ILLUMINACLIP:TruSeq3-PE.fa:2:30:10 \
$ SLIDINGWINDOW:4:15 \
$ MINLEN:75

# Clean up environment
$ module unload trimmomatic
```
Note: The above script will only work if all necessary files (input fastq and TruSeq3-PE.fa) are in the local directory from which you are running the script.
Save the script by pressing ^x, then y, then return. To submit the script as a slurm job, 
```bash
$ sbatch sbatch.trimmomatic
```
To check on job status, use
```bash
$ squeue -u yc1201
```
## Visualize the Trimmed FastQ files in FastQC Again to Ensure Quality Improvement
To do this, run FastQC on the newly generated _paired.fq.gz files to confirm that sequence quality has improved and adapters were removed.
```bash
# Enter interactive mode on a compute node
$ srun --pty bash

# Create an output directory for the post-trimming QC reports
$ mkdir -p /home/yc1201/genome_alignment_kcs1/results/qc_trimmed

# Load FastQC module
$ module load fastqc

# Run FastQC only on the paired-end trimmed files
$ fastqc -o /home/yc1201/genome_alignment_kcs1/results/qc_trimmed/ /home/yc1201/genome_alignment_kcs1/results/trimmed/*_paired.fq.gz
```
```bash
# Download FastQC reports for visualization
# Create local directory for reports
$ mkdir -p ~/Downloads/kcs1_fastqc_trimmed

#Download HTML files from the HPC
$ gcloud compute scp m12-controller:/home/yc1201/genome_alignment_kcs1/results/qc_trimmed/*.html ~/Downloads/kcs1_fastqc_trimmed
```

## Align Reads to Reference Genome
Align the reads in the FASTQ file to a reference genome that includes the gene of interest (in this case, KCS1).
Get NCBI reference genome fna file and upload to HPC: GCF_000182965.3_ASM18296v3_genomic.fna.gz
```bash
# Navigate to folder for reference genome on local computer
$ cd /home/yc1201/genome_alignment_kcs1/data/reference

# Download the genome from the official NCBI FTP server
$ wget https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_000182965.3/

# Decompress the genome file
$ gunzip GCF_000182965.3_ASM18296v3_genomic.fna.gz

# Enter interactive mode on compute node to build the index
$ srun --pty bash

# Load Bowtie2 module
$ module load bowtie2

# Build Bowtie2 index
$ bowtie2-build /home/yc1201/genome_alignment_kcs1/data/reference/GCF_000182965.3_ASM18296v3_genomic.fna /home/yc1201/genome_alignment_kcs1/data/reference/ref

$ echo "Reference genome indexing complete."
```
Write and run the bowtie script by submitting a slurm job.
```bash
# To write slurm script in window
$ nano sbatch.bowtie
```
In the window, write the slurm script.
```bash
#!/bin/bash
#SBATCH --job-name=bowtie_kcs1
#SBATCH --output=/home/yc1201/genome_alignment_kcs1/slurm/%x.o%j
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=erz12@georgetown.edu
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --time=06:00:00
#SBATCH --mem=8G
#SBATCH --array=6-8

# Establish base directory paths
$ BASE_DIR="/home/yc1201/genome_alignment_kcs1"
$ TRIM_DIR="${BASE_DIR}/results/trimmed"
$ ALIGN_DIR="${BASE_DIR}/results/aligned"

# Ensure alignment output folder exists
$ mkdir -p ${ALIGN_DIR}

# Map the SLURM Array ID to actual KCS1 sample names
$ if [ ${SLURM_ARRAY_TASK_ID} -eq 6 ]; then
    SAMPLE="Kcs1Restore6_S235"
$ elif [ ${SLURM_ARRAY_TASK_ID} -eq 7 ]; then
    SAMPLE="Kcs1Restore7_S236"
$ elif [ ${SLURM_ARRAY_TASK_ID} -eq 8 ]; then
    SAMPLE="Kcs1Restore8_S237"
$ fi

# Define input, reference, and output variables
$ READ1="${TRIM_DIR}/${SAMPLE}_R1_paired.fq.gz"
$ READ2="${TRIM_DIR}/${SAMPLE}_R2_paired.fq.gz"
$ REFERENCE_INDEX="${BASE_DIR}/data/reference/ref"
$ OUTPUT_FILE="${ALIGN_DIR}/${SAMPLE}.sam"

# Run Bowtie2 alignment (using 4 threads to speed it up)
$ module load bowtie2/2.5.3
$ bowtie2 -p 4 -x $REFERENCE_INDEX -1 $READ1 -2 $READ2 -S $OUTPUT_FILE

$ module unload bowtie2
```
Save the script by pressing ^x, then y, then return. To submit the script as a slurm job, 
```bash
$ sbatch sbatch.bowtie
```
To check on job status, use
```bash
$ squeue -u yc1201
```

## Extract Reads Mapping to the Gene
Because raw SAM files take up massive amounts of storage space, we convert them into compressed binary BAM files, sort them by genomic coordinates, and build an index for fast downstream processing and gene extraction.
First, sort and index SAM files.
```bash
# Enter interactive mode on compute node
$ srun --pty bash

# Load samtools module
$ module load samtools

# Navigate to alignment directory
$ cd /home/yc1201/genome_alignment_kcs1/results_aligned

# Candidate 6
# Convert SAM files to BAM
$ samtools view -S -b Kcs1Restore6_S235.sam > Kcs1Restore6_S235.bam
# Sort BAM by coordinate
$ samtools sort Kcs1Restore6_S235.bam -o Kcs1Restore6_S235.srt.bam
# Index the sorted BAM (generates Kcs1Restore6_S235.srt.bam.bai)
$ samtools index Kcs1Restore6_S235.srt.bam

# Candidate 7
# Convert SAM files to BAM
$ samtools view -S -b Kcs1Restore7_S236.sam > Kcs1Restore7_S236.bam
# Sort BAM by coordinate
$ samtools sort Kcs1Restore7_S236.bam -o Kcs1Restore7_S236.srt.bam
# Index the sorted BAM (generates Kcs1Restore7_S236.srt.bam.bai)
$ samtools index Kcs1Restore7_S236.srt.bam

# Candidate 8
# Convert SAM files to BAM
$ samtools view -S -b Kcs1Restore8_S237.sam > Kcs1Restore8_S237.bam
# Sort BAM by coordinate
samtools sort Kcs1Restore8_S237.bam -o Kcs1Restore8_S237.srt.bam
# Index the sorted BAM (generates Kcs1Restore8_S237.srt.bam.bai)
samtools index Kcs1Restore8_S237.srt.bam
```
To extract reads mapping to the gene, make reduced bams for only chromosome R (where KCS1 is located).
Chromosome R: NC_032096.1
```bash
# Extract chromosome R mapped reads
# Enter interactive mode on a compute node
$ srun --pty bash

# Load the samtools module
$ module load samtools

# Navigate to alignment directory
$ cd /home/yc1201/genome_alignment_kcs1/results/aligned

# Candidate 6
$ samtools view -h -b -o Kcs1Restore6_S235.chrR.bam Kcs1Restore6_S235.srt.bam NC_032096.1
# Sort and index Chromosome R files
$ samtools sort Kcs1Restore6_S235.chrR.bam -o Kcs1Restore6_S235.chrR.srt.bam
$ samtools index Kcs1Restore6_S235.chrR.srt.bam

# Candidate 7
$ samtools view -h -b -o Kcs1Restore7_S236.chrR.bam Kcs1Restore7_S236.srt.bam NC_032096.1
# Sort and index Chromosome R files
$ samtools sort Kcs1Restore7_S236.chrR.bam -o Kcs1Restore7_S236.chrR.srt.bam
$ samtools index Kcs1Restore7_S236.chrR.srt.bam

# Candidate 8
$ samtools view -h -b -o Kcs1Restore8_S237.chrR.bam Kcs1Restore8_S237.srt.bam NC_032096.1
# Sort and index Chromosome R files
$ samtools sort Kcs1Restore8_S237.chrR.bam -o Kcs1Restore8_S237.chrR.srt.bam
$ samtools index Kcs1Restore8_S237.chrR.srt.bam

# Downloading files to local cumputer
$ mkdir -p ~/Downloads/kcs1_alignmentR_results
$ gcloud compute scp m12-controller:/home/yc1201/genome_alignment_kcs1/results/aligned/Kcs1Restore*.chrR.srt.bam* ~/Downloads/kcs1_alignmentR_results/
```

Since we integrated KCS1 at a neutral locus on chromosome 5, make bam files corresponding to just chromosome 5.
Chromosome 5: NC_032093.1
```bash
# Extract chromosome 5 mapped reads
# Enter interactive mode on a compute node
$ srun --pty bash

# Load the samtools module
$ module load samtools

# Navigate to alignment directory
$ cd /home/yc1201/genome_alignment_kcs1/results/aligned

# Candidate 6
$ samtools view -h -b -o Kcs1Restore6_S235.chr5.bam Kcs1Restore6_S235.srt.bam NC_032096.1
# Sort and index Chromosome 5 files
$ samtools sort Kcs1Restore6_S235.chr5.bam -o Kcs1Restore6_S235.chr5.srt.bam
$ samtools index Kcs1Restore6_S235.chr5.srt.bam

# Candidate 7
$ samtools view -h -b -o Kcs1Restore7_S236.chr5.bam Kcs1Restore7_S236.srt.bam NC_032096.1
# Sort and index Chromosome 5 files
$ samtools sort Kcs1Restore7_S236.chr5.bam -o Kcs1Restore7_S236.chr5.srt.bam
$ samtools index Kcs1Restore7_S236.chr5.srt.bam

# Candidate 8
$ samtools view -h -b -o Kcs1Restore8_S237.chr5.bam Kcs1Restore8_S237.srt.bam NC_032096.1
# Sort and index Chromosome 5 files
$ samtools sort Kcs1Restore8_S237.chr5.bam -o Kcs1Restore8_S237.chr5.srt.bam
$ samtools index Kcs1Restore8_S237.chr5.srt.bam

# Downloading files to local cumputer
$ mkdir -p ~/Downloads/kcs1_alignment5_results
$ gcloud compute scp m12-controller:/home/yc1201/genome_alignment_kcs1/results/aligned/Kcs1Restore*.chr5.srt.bam* ~/Downloads/kcs1_alignment5_results/
```

## Analyze Extracted Reads
Download Integrated Genomics Viewer (IGV) on your local computer. In the top left menu, click "Genomes" then "Load genome from file."
To visualize alignments, load the correct genome from the downloaded file on your computer (GCF_000182965.3_ASM18296v3_genomic.fna). (Note: IGV will automatically detect the accompanying ".fai" index file as long as it is in the same folder).

Make a desktop folder with the sorted bam (srt.bam) and indexed bam (srt.bam.bai) files and upload them to IGV to inspect the alignments. In IGV, click "File" then "Load from file."
Select all 3 "*.srt.bam" files at once to load them as individual tracks.

To check for proper integration, paste the integration coordinates into the IGV locus search bar. Do not include commas.
Zoom into KCS1 location (chromosome R): NC_032096.1:1,136,463-1,135,405 --> verifies original deletion
Zoom into kcs1 location (chromosome 5): NC_032096.1:1136395-1135337 --> verifies successful transformation/restoration
Check for overlapping reads at KCS1 for all candidates.
