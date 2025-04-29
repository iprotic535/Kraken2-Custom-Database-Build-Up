# Building a Custom Kraken2 Database from NCBI Genomes

## Overview
This guide explains how to:
- Download genomes using NCBI `datasets` CLI.
- Format genome headers for Kraken2.
- Build a custom Kraken2 database.
- Run Kraken2 classification.

---

## Requirements

- `kraken2`
- `datasets` (NCBI Datasets CLI)
- `unzip`
- (Optional) `find`, `wget`, `bash`

### Install if needed
```bash
# Install Kraken2
conda install -c bioconda kraken2

# Install NCBI Datasets CLI
conda install -c conda-forge ncbi-datasets-cli

# Install unzip utility
sudo apt install unzip
```

---

## Step 1: Download Genomes from NCBI by Taxon ID

Example: Download all genomes for **Taxon ID 35581** (Myxozoa)

```bash
# Create a working directory for downloads
mkdir -p ~/datasets_download
cd ~/datasets_download

# Download dehydrated metadata for TaxID 35581 (only metadata, not full genomes)
datasets download genome taxon 35581 --dehydrated --filename taxid_35581.zip

# Unzip the downloaded metadata package
unzip taxid_35581.zip -d taxid_35581

# Rehydrate to actually download all genome FASTA files (.fna)
datasets rehydrate --directory taxid_35581
```

Result: Genomes will be stored under `taxid_35581/ncbi_dataset/data/`

---

## Step 2: Format Genomes for Kraken2

Kraken2 requires each sequence header to be in the format `>kraken:taxid|<TAXID>|OriginalHeader`.

To reformat all `.fna` files without losing the original header:

```bash
# Find all .fna files and prepend Kraken2 taxid format to existing headers
find ~/datasets_download/taxid_35581 -type f -name "*.fna" -exec sed -i 's/^>/>kraken:taxid|35581|/' {} \;
```

This command will modify each header line by inserting the Kraken2-required prefix while keeping the original sequence identifier.

---

## Step 3: Create Kraken2 Database Structure

```bash
# Create a new directory for the Kraken2 database
mkdir -p ~/kraken_custom_db

# Download NCBI taxonomy data into the database directory
kraken2-build --download-taxonomy --db ~/kraken_custom_db

```

---

## Step 4: Add Genomes to Kraken2 Database

```bash
# Add all formatted genome files to the Kraken2 database library
find ~/datasets_download/taxid_35581/ -name "*.fna" -exec kraken2-build --add-to-library {} --db ~/kraken_custom_db \;
```

### Method 2: Using `k2 add-to-library`
```bash
# Alternatively, add all formatted genome files using k2 add-to-library, which is a new wrapper introduced by the developers 
for fasta in $(find ~/datasets_download/taxid_35581/ -name "*.fna"); do
    k2 add-to-library --db ~/kraken_custom_db --threads 15 --file "$fasta" #k2 has the new --threads option, which is handy when adding a large number of genomes
done
```

This step prepares the genome sequences for indexing.

---

## Step 5: Build the Kraken2 Database

```bash
# Build the Kraken2 database using 8 CPU threads
kraken2-build --build --db ~/kraken_custom_db --threads 8
```

### Method 2: Using `k2 build`
```bash
# Build the Kraken2 database using 15 CPU threads
k2 build --db ~/kraken_custom_db --threads 15
```

This process will create the database files (hash table, taxonomy, options).

---

## Step 6: Run Kraken2 on Sample Data

Assuming you have a sample file `sample.fastq`:

```bash
# Classify sequences in the sample file using the custom database
kraken2 --db ~/kraken_custom_db --threads 8 --report sample_report.txt --output sample_output.txt sample.fastq
```

- `sample_report.txt`: summary table of taxonomic matches
- `sample_output.txt`: classification for each individual read

---


## ✅ Summary

You now have:
- Downloaded NCBI genomes by TaxID using `datasets`
- Formatted the genomes for Kraken2 without losing sequence identifiers
- Built a custom Kraken2 classification database
- Classified sample data and generated reports

---

## 📚 References

- [NCBI Datasets CLI Documentation](https://www.ncbi.nlm.nih.gov/datasets/docs/v2/)
- [Kraken2 GitHub Repository](https://github.com/DerrickWood/kraken2)

---

_Authored by: Ismam Ahmed Protic_  
_Date: April 2025_

