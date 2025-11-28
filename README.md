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
conda activate miniforg/envs/qiime2-amplicon-2023.9.

# Import sequences
## Before running any analysis, QIIME2 needs raw FASTQ files converted into a .qza artifact.
This step imports your paired-end sequencing data in Casava 1.8 format (the standard Illumina naming structure) and packages it as a QIIME2 artifact (SampleData[PairedEndSequencesWithQuality]), which can then be used for demultiplexing, quality visualization, and denoising.

This ensures:
QIIME2 understands the format of your sequence files
Read orientation and metadata are correctly recognized
Downstream tools like DADA2 can process the data

```bash
qiime tools import \
--type 'SampleData[PairedEndSequencesWithQuality]' \
--input-path fastq_files \
--input-format CasavaOneEightSingleLanePerSampleDirFmt \
--output-path allseq.qza

# Convert from qza to qzv
qiime demux summarize \
--i-data allseq.qza \
--o-visualization allseq.qzv

# Denoising, trimming and joining 
time qiime dada2 denoise-paired \
--i-demultiplexed-seqs allseq.qza \
--p-trim-left-f 10 --p-trunc-len-f 240 \
--p-trim-left-r 10 --p-trunc-len-r 240 \
--p-n-threads 0 \
--o-table table-dada.qza \
--o-representative-sequences repseqs-dada.qza \
--o-denoising-stats stats-data.qza

# Visualise denoised, trimmed file
qiime metadata tabulate \
--m-input-file table-dada.qza \ 
--o-visualization table-data.qzv 

qiime metadata tabulate \      
--m-input-file repseqs-dada.qza \
--o-visualization repseqs-data.qzv 

qiime metadata tabulate \
--m-input-file stats-data.qza \
--o-visualization ./qzv-files/stats-data.qzv

# Table summary
qiime feature-table summarize \
  --i-table table-dada.qza \
  --o-visualization table-summary.qzv

# Taxonomy classification using rescript classifier
time qiime feature-classifier classify-sklearn \ 
  --i-classifier silva-138-ssu-nr99_rescript_classifier_shiva.qza \
  --i-reads repseqs-dada.qza \
  --o-classification rc-rescript-taxonomy.qza

# Visualise rescript taxonomy
qiime metadata tabulate \
  --m-input-file rc-rescript-taxonomy.qza \
  --o-visualization rc-rescript-taxonomy.qzv

# Taxonomy-Box-bar-plot
qiime taxa barplot \
  --i-table table.qza \
  --i-taxonomy taxonomy.qza \
  --m-metadata-file meta-data.tsv \
  --o-visualization taxa-bar-plots.qzv

# Filtering taxa table
time qiime taxa filter-table \
  --i-table table-dada.qza \ 
  --i-taxonomy rc-rescript-taxonomy.qza \ 
  --p-mode contains \     
  --p-include p__ \
  --p-exclude 'd__Archaea, Unassigned' \
  --o-filtered-table filtered-table-dada.qza

# filtered table summary
qiime feature-table summarize \
  --i-table ./filtered-qza/filtered-table-dada.qza \
  --o-visualization ./filtered-qzv/filtered-table-summary.qzv

# Filtering taxa repseqs
qiime feature-table filter-seqs \
  --i-data repseqs-dada.qza \
  --i-table filtered-table-dada.qza \ 
  --o-filtered-data filtered-repseq-dada.qza

# Filtered Taxonomy classification and visualization
time qiime feature-classifier classify-sklearn \
  --i-classifier ./qza-files/silva-138-ssu-nr99_rescript_classifier_shiva.qza \
  --i-reads ./filtered-qza/filtered-repseq-dada.qza \ 
  --o-classification ./filtered-qza/filtered-taxonomy.qza

qiime metadata tabulate \
  --m-input-file rc-rescript-taxonomy.qza \
  --o-visualization rc-rescript-taxonomy.qzv


## 📂 Output Files
- `table-dada.qza` – feature table after DADA2
- `repseqs-dada.qza` – representative sequences
- `rc-rescript-taxonomy.qza` – taxonomy assignments
- `filtered-table-dada.qza` – filtered table excluding Archaea & unassigned
- `filtered-taxonomy.qza` – final taxonomy output

## Citation
- Callahan BJ et al., Nat Methods, 2016 – **DADA2**.

## 👤 Author
**G. Sivasubramaniyan**  
Assistant Professor & PhD Research Scholar, Department of Microbiology  
Sri Ramachandra Institute of Higher Education and Research (SRIHER), Chennai, India  
Email id: shivaviro24@gmail.com, sivasubramaniyan@sriramachandra.edu.in

**Acknowledgment:**  Workflow guidance provided by a collaborator from Christian Medical College Vellore.



