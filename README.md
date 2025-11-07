# RNASeq_Analysis
Trying to analyze the raw readouts and allign them first followed by using DESeq2
# RNA-seq Analysis in Google Colab

This document summarizes the steps taken to set up a Google Colab environment for RNA-seq analysis using STAR and DESeq2, including troubleshooting steps for common issues.

## Setup

1.  **Set up R environment and Install DESeq2:**
    We installed necessary R packages, including DESeq2, using `rpy2` to run R code within the Python notebook.

    ```python
    %load_ext rpy2.ipython
    ```

    ```R
    install.packages("devtools")
    install.packages("BiocManager")
    BiocManager::install("DESeq2")
    ```

    We verified the installation:

    ```R
    library(DESeq2)
    packageVersion("DESeq2")
    ```

2.  **Install Bioinformatics Tools (STAR):**
    We downloaded pre-compiled STAR binaries as `apt-get` did not have the package.

    ```bash
    %%bash
    # Define the download URL for STAR binaries (example for Linux)
    STAR_URL="https://github.com/alexdobin/STAR/archive/refs/tags/2.7.10a.tar.gz"
    wget $STAR_URL -O STAR.tar.gz
    mkdir STAR_bin
    tar -xzf STAR.tar.gz -C STAR_bin --strip-components=1
    STAR_EXEC="./STAR_bin/bin/Linux_x86_64/STAR" # Path to STAR executable
    ```

3.  **Data Preparation (Reference Genome):**
    Due to memory constraints with the full human genome, we decided to use human chromosome 22. We downloaded the FASTA and GTF files.

    ```bash
    %%bash
    mkdir -p genome_chr22/fasta genome_chr22/gtf
    FASTA_URL_CHR22="http://ftp.ensembl.org/pub/release-110/fasta/homo_sapiens/dna/Homo_sapiens.GRCh38.dna.chromosome.22.fa.gz"
    GTF_URL_CHR22="http://ftp.ensembl.org/pub/release-110/gtf/homo_sapiens/Homo_sapiens.GRCh38.110.gtf.gz"
    wget -O genome_chr22/fasta/Homo_sapiens.GRCh38.dna.chromosome.22.fa.gz $FASTA_URL_CHR22
    wget -O genome_chr22/gtf/Homo_sapiens.GRCh38.110.gtf.gz $GTF_URL_CHR22
    ```

    We then uncompressed these files as required by STAR.

    ```bash
    %%bash
    gunzip genome_chr22/fasta/Homo_sapiens.GRCh38.dna.chromosome.22.fa.gz
    gunzip genome_chr22/gtf/Homo_sapiens.GRCh38.110.gtf.gz
    ```

4.  **Build STAR Genome Index:**
    We built the STAR index for chromosome 22, adjusting the `genomeSAindexNbases` parameter as recommended by STAR for smaller genomes.

    ```bash
    %%bash
    GENOME_DIR="genome_index_chr22"
    GENOME_FASTA="genome_chr22/fasta/Homo_sapiens.GRCh38.dna.chromosome.22.fa"
    GENOME_GTF="genome_chr22/gtf/Homo_sapiens.GRCh38.110.gtf"
    NUM_THREADS=4
    STAR_EXEC="./STAR_bin/bin/Linux_x86_64/STAR"
    GENOME_SANBASES=11 # Recommended parameter for smaller genomes
    mkdir -p $GENOME_DIR
    $STAR_EXEC --runMode genomeGenerate \
        --genomeDir $GENOME_DIR \
        --genomeFastaFiles $GENOME_FASTA \
        --sjdbGTFfile $GENOME_GTF \
        --runThreadN $NUM_THREADS \
        --genomeSAindexNbases $GENOME_SANBASES
    ```

5.  **Data Access (RNA-seq Reads):**
    We mounted Google Drive to access RNA-seq data stored there.

    ```python
    from google.colab import drive
    drive.mount('/content/drive')
    ```

    We also extracted the contents of a main zip file containing the FASTQ files.

    ```bash
    %%bash
    ZIP_FILE_PATH="/content/drive/My Drive/RNA_seq_raw/Dayi_iNGN2_SMG1i.zip" # Replace with your actual path
    EXTRACT_DIR="extracted_rnaseq_data"
    mkdir -p $EXTRACT_DIR
    unzip "$ZIP_FILE_PATH" -d $EXTRACT_DIR
    ```

    *(Note: We encountered a zip file read error during extraction, which might require addressing the integrity of the original zip file.)*

## Sequence Alignment with STAR

We attempted to perform STAR alignment using the chromosome 22 index and the first sample's FASTQ files from the extracted data. We directed the output to Google Drive and specified a temporary directory on the Colab local disk to avoid FIFO file creation issues on the Drive mount.

```bash
%%bash
GENOME_DIR="genome_index_chr22"
READS_R1="extracted_rnaseq_data/iNGN2-1_R1_001.fastq.gz" # Corrected filename
READS_R2="extracted_rnaseq_data/iNGN2-1_R2_001.fastq.gz" # Corrected filename
OUTPUT_DIR="/content/drive/My Drive/star_alignment_results" # Replace with your actual desired path in Google Drive
TMP_DIR="/tmp/star_tmp"
NUM_THREADS=4
STAR_EXEC="./STAR_bin/bin/Linux_x86_64/STAR"
mkdir -p "$OUTPUT_DIR"
rm -rf "$TMP_DIR" # Remove temporary directory before creating
mkdir -p "$TMP_DIR" # Create temporary directory on local disk
$STAR_EXEC --runMode alignReads \
    --genomeDir $GENOME_DIR \
    --readFilesIn $READS_R1 $READS_R2 \
    --readFilesCommand zcat \
    --runThreadN $NUM_THREADS \
    --outFileNamePrefix "${OUTPUT_DIR}/sample_1_" \
    --outSAMtype BAM SortedByCoordinate \
    --outSAMunmapped None \
    --outSAMattributes Standard \
    --outTmpDir "$TMP_DIR" # Specify temporary directory on local disk
