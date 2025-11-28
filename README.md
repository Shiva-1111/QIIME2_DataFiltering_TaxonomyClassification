# QIIME2 Workflow: Data Filtering & Taxonomy Classification

This repository documents the QIIME2 workflow used for paired-end 16S rRNA data processing, quality filtering, DADA2 denoising, and taxonomy classification using a SILVA 138 RESCRIPt-trained classifier.

---

## Steps Overview
1. **Activate environment**
2. **Import sequences & summarize quality**
3. **Denoise using DADA2**
4. **Generate feature tables**
5. **Classify taxonomy with pre-trained SILVA classifier**
6. **Filter Archaea & unassigned reads**
7. **Visualize taxa bar plots**

---

## ACTIVATE CONDA
bash
# Activate QIIME2 environment
`conda activate miniforg/envs/qiime2-amplicon-2023.9`

# 1. Import sequences
## Before running any analysis, QIIME2 needs raw FASTQ files converted into a .qza artifact.
#### This step imports your paired-end sequencing data in Casava 1.8 format (the standard Illumina naming structure) and packages it as a QIIME2 artifact (SampleData[PairedEndSequencesWithQuality]), which can then be used for demultiplexing, quality visualization, and denoising. This ensures: 1) QIIME2 understands the format of your sequence files. 2) Read orientation and metadata are correctly recognized. 3) Downstream tools like DADA2 can process the data

```bash
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path fastq_files \
  --input-format CasavaOneEightSingleLanePerSampleDirFmt \
  --output-path allseq.qza
```

# 1.1 Convert from qza to qzv
### Converts the .qza file into a .qzv file, which can be opened in QIIME2 View to inspect read quality before denoising.
```bash
qiime demux summarize \
--i-data allseq.qza \
--o-visualization allseq.qzv
```

# 2. Denoising, trimming and joining 
### This step uses **DADA2** to:
- Trim low-quality bases at the start of each read  
- Truncate reads to a fixed length based on quality  
- Denoise reads (error-model based correction)  
- Merge forward and reverse reads  
- Remove chimeras  
- Produce:
  - A **feature table** (`table-dada.qza`)
  - **Representative sequences** (`repseqs-dada.qza`)
  - **Denoising statistics** (`stats-data.qza`)

Key points:
- `--p-trim-left-f 10` / `--p-trim-left-r 10`  
  → Remove the first 10 bases from forward and reverse reads (often low-quality or primer region).
- `--p-trunc-len-f 240` / `--p-trunc-len-r 240`  
  → Truncate reads to 240 bp based on quality decay and expected overlap for merging.
- `--p-n-threads 0`  
  → Use all available CPUs for faster processing.
- `table-dada.qza`  
  → ASV count table used in all downstream analyses.
- `repseqs-dada.qza`  
  → One representative sequence per ASV (used for taxonomy and tree building).
- `stats-data.qza`  
  → Summary of reads kept/removed at each step (important for QC).

```bash
time qiime dada2 denoise-paired \
--i-demultiplexed-seqs allseq.qza \
--p-trim-left-f 10 --p-trunc-len-f 240 \
--p-trim-left-r 10 --p-trunc-len-r 240 \
--p-n-threads 0 \
--o-table table-dada.qza \
--o-representative-sequences repseqs-dada.qza \
--o-denoising-stats stats-data.qza`
```


# 3. Visualise denoised, trimmed file
### After running DADA2, QIIME2 generates three important output files:
- **Feature table (`table-dada.qza`)** — counts of ASVs per sample  
- **Representative sequences (`repseqs-dada.qza`)** — the ASV sequences  
- **Denoising statistics (`stats-data.qza`)** — details on filtering, merging, and chimera removal  

These `.qza` files are converted into `.qzv` visualizations so they can be viewed using **QIIME2 View**.

Purpose of each visualization:
- `table-data.qzv` → Shows number of features and read depth per sample  
- `repseqs-data.qzv` → Lists all ASVs and their sequences  
- `stats-data.qzv` → Shows how many reads passed each DADA2 step (quality, merging, chimera filtering)

```bash
qiime metadata tabulate \
--m-input-file table-dada.qza \ 
--o-visualization table-data.qzv

qiime metadata tabulate \      
--m-input-file repseqs-dada.qza \
--o-visualization repseqs-data.qzv

qiime metadata tabulate \
--m-input-file stats-data.qza \
--o-visualization ./qzv-files/stats-data.qzv
```

# 4. Table summary
### This step generates a summary of the **ASV feature table** created by DADA2.  
The output `.qzv` file provides important information about sequencing depth and sample quality.

What this visualization shows:
- Total number of ASVs  
- Number of ASVs per sample  
- Read depth distribution (min, max, median, quartiles)  
- A table of sample-wise sequencing depths  
- Helps decide **sampling depth** for alpha/beta diversity

```bash
qiime feature-table summarize \
  --i-table table-dada.qza \
  --o-visualization table-summary.qzv
```

# 5. Taxonomy classification using rescript classifier
### 
This step assigns taxonomy to each ASV using a **Naive Bayes classifier** trained on the  
**SILVA 138 NR99** reference database (prepared through RESCRIPt).

Purpose of this step:
- Matches each representative ASV sequence to the closest reference sequence  
- Provides taxonomy from Kingdom → Species (depending on classifier resolution)  
- Generates a `.qza` taxonomy file for downstream visualization and filtering  

Key inputs:
- `silva-138-ssu-nr99_rescript_classifier_shiva.qza` → Your custom-trained RESCRIPt classifier  
- `repseqs-dada.qza` → ASV sequences generated by DADA2  

Output:
- `rc-rescript-taxonomy.qza` → Taxonomic assignments for each ASV

```bash
time qiime feature-classifier classify-sklearn \ 
  --i-classifier silva-138-ssu-nr99_rescript_classifier_shiva.qza \
  --i-reads repseqs-dada.qza \
  --o-classification rc-rescript-taxonomy.qza
```

# 5.1 Visualise rescript taxonomy
### This step converts the taxonomy `.qza` file into a `.qzv` visualization.  
The `.qzv` file allows you to:

- View taxonomic assignments for each ASV  
- Inspect confidence scores  
- Check how many ASVs were assigned at each taxonomic rank  
- Identify unassigned or low-confidence classifications  

This is important for verifying whether the classifier worked properly.
```bash
qiime metadata tabulate \
  --m-input-file rc-rescript-taxonomy.qza \
  --o-visualization rc-rescript-taxonomy.qzv
```

# 5.2 Taxonomy-Box-bar-plot
### This visualization helps you:

- Compare microbial composition across groups  
- See dominant phyla/genera  
- Identify patterns of **Taxonomic** profiles  
- Quickly assess overall microbiome structure  

**Inputs:**
- `table.qza` → Feature table  
- `taxonomy.qza` → Taxonomic assignments  
- `meta-data.tsv` → Sample metadata  

**Output:**
- `taxa-bar-plots.qzv` → Interactive bar plot
```bash
qiime taxa barplot \
  --i-table table.qza \
  --i-taxonomy taxonomy.qza \
  --m-metadata-file meta-data.tsv \
  --o-visualization taxa-bar-plots.qzv
```

# 5.3 Filtering taxa table
### This step filters the feature table to retain only bacterial sequences.  
We exclude **Archaea** and **Unassigned** ASVs because:

- They may not be relevant to human gut 16S studies  
- They often come from low-confidence or off-target amplification  
- Removing them improves downstream diversity and differential abundance analyses  

**`--p-include p__`** → Keep only features with a phylum-level assignment  
**`--p-exclude 'd__Archaea, Unassigned'`** → Remove Archaea & unassigned reads  
```bash
time qiime taxa filter-table \
  --i-table table-dada.qza \ 
  --i-taxonomy rc-rescript-taxonomy.qza \ 
  --p-mode contains \     
  --p-include p__ \
  --p-exclude 'd__Archaea, Unassigned' \
  --o-filtered-table filtered-table-dada.qza
```

# 5.4 Filtered table summary
### This step generates a summary of the **filtered ASV table** after removing Archaea and unassigned reads.

This visualization shows:
- Number of samples retained  
- Number of ASVs after filtering  
- Minimum, median, and maximum sequencing depth  
- Read depth distribution across samples  

Useful for confirming that filtering did not remove too many samples or reads.
```bash
qiime feature-table summarize \
  --i-table ./filtered-qza/filtered-table-dada.qza \
  --o-visualization ./filtered-qzv/filtered-table-summary.qzv
```

# 5.5 Filtering taxa repseqs
### This step filters the representative sequences file to keep only those ASVs present in the filtered feature table.

Why this is needed:
- Ensures consistency between table and sequences
- Prevents mismatched taxonomy or phylogeny
- Only filtered ASVs are used for tree building, diversity, and downstream analysis
```bash
qiime feature-table filter-seqs \
  --i-data repseqs-dada.qza \
  --i-table filtered-table-dada.qza \ 
  --o-filtered-data filtered-repseq-dada.qza
```

# 5.6 Filtered Taxonomy classification and visualization
### We classify taxonomy again using the filtered representative sequences to ensure that taxonomy corresponds exactly to the filtered ASVs.

Outputs:

`filtered-taxonomy.qza` → Taxonomy for filtered ASVs
`rc-rescript-taxonomy.qzv` → Visual summary of taxonomy

```bash
time qiime feature-classifier classify-sklearn \
  --i-classifier ./qza-files/silva-138-ssu-nr99_rescript_classifier_shiva.qza \
  --i-reads ./filtered-qza/filtered-repseq-dada.qza \ 
  --o-classification ./filtered-qza/filtered-taxonomy.qza
```

# This step converts file `.qza` to `.qzv` to visualise using **QIIME2** View
```bash
qiime metadata tabulate \
  --m-input-file rc-rescript-taxonomy.qza \
  --o-visualization rc-rescript-taxonomy.qzv
```


## 📂 Output Files
- `table-dada.qza` – feature table after DADA2
- `repseqs-dada.qza` – representative sequences
- `rc-rescript-taxonomy.qza` – taxonomy assignments
- `filtered-table-dada.qza` – filtered table excluding Archaea & unassigned
- `filtered-taxonomy.qza` – final taxonomy output

## Citation
- Yet to add

## 👤 Author
**G. Sivasubramaniyan**  
Assistant Professor & PhD Research Scholar, Department of Microbiology  
Sri Ramachandra Institute of Higher Education and Research (SRIHER), Chennai, India  
Email id: shivaviro24@gmail.com, sivasubramaniyan@sriramachandra.edu.in

**Acknowledgment:**  Workflow guidance provided by a collaborator from Christian Medical College Vellore.



