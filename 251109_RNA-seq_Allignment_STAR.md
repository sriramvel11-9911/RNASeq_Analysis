# RNA-seq Alignment and Counting with STAR and featureCounts (Chromosome 22 Validation)

This document outlines the step-by-step process to align RNA-seq fastq files to human chromosome 22 using STAR and generate a count matrix using featureCounts within a Windows Subsystem for Linux (WSL) Ubuntu environment.

---

## 1. Prepare your environment

Ensure you have the necessary tools installed in your Ubuntu environment (STAR, samtools, subread/featureCounts).

**Action:** Copy and paste the following commands into your Ubuntu terminal and press Enter after each line:

    mkdir -p fasta gtf
    FASTA_URL_CHR22="http://ftp.ensembl.org/pub/release-110/fasta/homo_sapiens/dna/Homo_sapiens.GRCh38.dna.chromosome.22.fa.gz"
    GTF_URL_CHR22="http://ftp.ensembl.org/pub/release-110/gtf/homo_sapiens/Homo_sapiens.GRCh38.110.gtf.gz"

    echo "Downloading chromosome 22 fasta..."
    wget -O fasta/Homo_sapiens.GRCh38.dna.chromosome.22.fa.gz $FASTA_URL_CHR22

    echo "Downloading chromosome 22 GTF..."
    wget -O gtf/Homo_sapiens.GRCh38.110.gtf.gz $GTF_URL_CHR22

    echo "Downloads started. This will take a while."

    gunzip genome_chr22/fasta/Homo_sapiens.GRCh38.dna.chromosome.22.fa.gz
    gunzip genome_chr22/gtf/Homo_sapiens.GRCh38.110.gtf.gz

    # Navigate to the directory where you want the index to be created (e.g., your home directory)
    cd ~

# 2. Now, run the STAR command. Adjust paths if necessary.
    GENOME_DIR="./STAR_index_chr22" # Directory to store the index (relative to current location)
    GENOME_FASTA="path/to/your/genome_chr22/fasta/Homo_sapiens.GRCh38.dna.chromosome.22.fa" # Path to your unzipped .fa file
    GENOME_GTF="path/to/your/genome_chr22/gtf/Homo_sapiens.GRCh38.110.gtf"       # Path to your unzipped .gtf file
    NUM_THREADS=8 # Adjust based on your CPU cores
    GENOME_SANBASES=11 # Recommended for smaller genomes

    mkdir -p $GENOME_DIR # Ensure the output directory exists

    STAR --runMode genomeGenerate \
        --genomeDir $GENOME_DIR \
        --genomeFastaFiles $GENOME_FASTA \
        --sjdbGTFfile $GENOME_GTF \
        --runThreadN $NUM_THREADS \
        --genomeSAindexNbases $GENOME_SANBASES

    # Define variables for clarity (adjust paths as needed)
    STAR_INDEX_DIR="~/STAR_index_chr22" # Path to your STAR genome index directory (using home shortcut)

    # Define variables for your paired-end fastq files
    # Update these paths to the actual location of your unzipped fastq files (.fastq) on your Windows drive
    # Example path for a file on your C: drive: /mnt/c/Users/YourUsername/YourDataFolder/your_sample_R1.fastq
    READS_R1="/path/to/your/fastq_files/your_sample_R1.fastq"
    READS_R2="/path/to/your/fastq_files/your_sample_R2.fastq"

# 3. Sample Name and Output
    # Define a sample name for the output
    SAMPLE_NAME="your_sample" # Use a descriptive name for this sample pair

    #  Define the output directory for this sample
    OUTPUT_DIR="./alignment_results/${SAMPLE_NAME}" # Output directory within Ubuntu (relative to where you run this command)

    # Ensure the output directory exists in your Ubuntu environment
    mkdir -p $OUTPUT_DIR

    echo "Starting alignment for sample: ${SAMPLE_NAME}"

# 4. Run STAR alignment
    # Removed --readFilesCommand zcat since files are unzipped (.fastq)
    STAR --runThreadN 8 \
    --genomeDir $STAR_INDEX_DIR \
    --readFilesIn $READS_R1 $READS_R2 \
    --outFileNamePrefix ${OUTPUT_DIR}/${SAMPLE_NAME}_ \
    --outSAMtype BAM SortedByCoordinate \
    --outSAMattributes Standard

    echo "Finished alignment for sample: ${SAMPLE_NAME}"
    echo "----------------------------------------"

    # Repeat this block for each sample, updating READS_R1, READS_R2, and SAMPLE_NAME.
    # Or use a loop for multiple samples as previously discussed.

#-----------------------------------------------------------------------------------------
# Have completed until the step above
#-----------------------------------------------------------------------------------------

# Define variables (adjust paths as needed)
    GTF_FILE="path/to/your/genome_chr22/gtf/Homo_sapiens.GRCh38.110.gtf" # Path to your unzipped GTF file

# Define the input BAM files - list all your aligned BAM files here
# Example for one sample:
    INPUT_BAM_FILES="./alignment_results/your_sample/your_sample_Aligned.sortedByCoord.out.bam"

# Example for multiple samples:
# INPUT_BAM_FILES="./alignment_results/sample1/sample1_Aligned.sortedByCoord.out.bam ./alignment_results/sample2/sample2_Aligned.sortedByCoord.out.bam"
# Add more BAM files separated by spaces for each sample you aligned.

# Define the output count matrix file name
    OUTPUT_COUNTS_FILE="./featureCounts_counts.txt"

    echo "Starting featureCounts..."

# Run featureCounts
    featureCounts -p \
    -t exon \
    -g gene_id \
    -a $GTF_FILE \
    -o $OUTPUT_COUNTS_FILE \
    $INPUT_BAM_FILES

    echo "Finished featureCounts."

    chmod +x rnaseq_pipeline_chr22.sh

    ./rnaseq_pipeline_chr22.sh

It's understandable to wonder why the alignment is taking a long time. While using a smaller reference like chromosome 22 is good for testing and requires less memory for the index compared to the whole genome, the *alignment speed* itself depends more on:

1.  **The total number of reads in your fastq files:** More reads means more work for STAR.
2.  **Your computer's processing power (CPU and RAM):** Alignment is computationally intensive. Limited resources will make it slower.
3.  **Disk speed:** Reading fastq files and writing BAM files can be bottlenecked by slow storage.

Using the **entire human genome** will require **significantly more memory** to build the index and will generally take **much longer** for both index generation and alignment compared to just chromosome 22, especially on systems with limited resources. So, switching to the whole genome will likely make the process *slower* and *more resource-intensive*, not faster for alignment itself (though it's necessary for aligning reads that map to other chromosomes).

Your current slow speed on chromosome 22 is more likely related to the size of your fastq data or your system's resources.

***

**Tips for Choosing a Reference Genome:**

When choosing a reference genome and annotations for RNA-seq analysis, consider the following:

1.  **Species:** Make sure the reference genome matches the species your RNA-seq data comes from. Human RNA-seq needs a human reference.
2.  **Assembly Version:** Use a widely accepted and stable assembly version (like GRCh38 for human). Consistency is key if you are comparing your data to other datasets. Ensure the GTF annotation file you use is built for the *exact same assembly version* as the fasta file.
3.  **Annotation Version:** Choose a reputable gene annotation source (like Ensembl or GENCODE). Use a recent, stable version of the annotations. Again, ensure compatibility with the genome assembly.
4.  **Completeness:** For full analysis, you'll need the entire genome assembly (all chromosomes, mitochondria, etc.) and comprehensive annotations. Chromosome 22 is only for testing or specific analyses.
5.  **Source Consistency:** Ideally, download the fasta and GTF files from the same source (e.g., both from Ensembl or both from GENCODE via a source like GDC or UCSC) to minimize compatibility issues.

***

Okay, let's proceed with deleting the current chromosome 22 reference files and index, and then provide commands to download the full human genome and annotations.

**Step 1: Delete the current chromosome 22 files and index**

Be very careful with the `rm -r` command, as it permanently deletes files and directories. Make sure you are targeting the correct folders.

Please copy and paste the following commands into your Ubuntu terminal. **Double-check the paths before running!**

# Navigate to your home directory or where these folders are located
    cd ~

# Remove the genome_chr22 directory (contains fasta and gtf)
# BE CAREFUL: This deletes everything inside this folder!
    rm -r genome_chr22

# Remove the STAR index directory for chromosome 22
# BE CAREFUL: This deletes everything inside this folder!
    rm -r STAR_index_chr22

    echo "Removed chromosome 22 reference files and STAR index."

Run these commands *only after confirming the paths are correct* and you are ready to delete those specific folders. Let me know once you've done this.

***

**Step 2: Download the entire human genome and annotations**

We will use `wget` again, but this time targeting the full human genome (GRCh38) and its annotations. I'll use Ensembl as a source, similar to your previous successful download method.

Please copy and paste the following commands into your Ubuntu terminal. This will download large files and may take a long time.

# Navigate to a directory where you want to store the full genome reference
# Example: Creating a 'genome_full' directory in your home folder
    mkdir ~/genome_full
    cd ~/genome_full

# Define URLs for the full human genome (GRCh38) and GTF from Ensembl
# These URLs are examples for release 110, you may want a more recent release
# Check ensembl.org/info/data/ftp/index.html for the latest release and correct paths
    FASTA_URL_GRCH38="http://ftp.ensembl.org/pub/release-110/fasta/homo_sapiens/dna/Homo_sapiens.GRCh38.dna.primary_assembly.fa.gz"
    GTF_URL_GRCH38="http://ftp.ensembl.org/pub/release-110/gtf/homo_sapiens/Homo_sapiens.GRCh38.110.gtf.gz"

    echo "Downloading full human genome fasta..."
    wget -O Homo_sapiens.GRCh38.dna.primary_assembly.fa.gz $FASTA_URL_GRCH38

    echo "Downloading full human genome GTF..."
    wget -O Homo_sapiens.GRCh38.110.gtf.gz $GTF_URL_GRCH38

    echo "Downloads started. This will take a while."

**Important:**

*   The URLs above are examples for Ensembl release 110. It's highly recommended to visit the Ensembl FTP site ([ensembl.org/info/data/ftp/index.html](ensembl.org/info/data/ftp/index.html)) to find the latest release and the exact paths for the `primary_assembly.fa.gz` and `.gtf.gz` files. Update the `FASTA_URL_GRCH38` and `GTF_URL_GRCH38` variables accordingly.
*   These are very large files. Make sure you have ample disk space before starting the download.

Run the deletion commands first (carefully!), then the download commands. Let me know once the downloads are complete. After that, the next step will be to build a new STAR index using these full genome files (which will take significantly longer and require more RAM than the chromosome 22 index).

***

Finally, regarding visual resources:

It can be challenging to find single YouTube videos that comprehensively cover both reference genome selection and the detailed internal workings of STAR annotation indexing. However, here are some types of resources that might help you visualize and understand these concepts:

*   **Bioinformatics Course Lectures:** Many universities and online platforms offer free bioinformatics courses on YouTube or Coursera/edX. Search for lectures on "RNA-seq alignment", "genome indexing", "gene annotation formats (GTF/GFF3)", and "mapping algorithms".
*   **STAR Documentation/Tutorials:** The official STAR manual and wiki often have diagrams or explanations of the index structure and alignment process.
*   **Blog Posts and Tutorials:** Many bioinformatics blogs or online tutorials walk through the steps of RNA-seq analysis with diagrams explaining different file formats and tool outputs.

While I can't provide live video content here, searching for these terms on YouTube or Google Scholar might yield useful visual explanations.

Let me know how the deletion and download steps go!