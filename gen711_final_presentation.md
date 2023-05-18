---
marp: true
theme: default
size: 16:9
paginate: true
footer: Kaleb
style: |
    section{
      justify-content: flex-start;
    }
---

# Bioinformatics Final Presentation
Kaleb Ducharme

---

# Outline

- Background Information
- Methods
- Results
- Final Remarks
- Bibliography

---

# Background Information

Original Data from:

###### Kang, DW., Adams, J.B., Gregory, A.C. et al. Microbiota Transfer Therapy alters gut ecosystem and improves gastrointestinal and autism symptoms: an open-label study. Microbiome 5, 10 (2017). https://doi.org/10.1186/s40168-016-0225-7

- tested how fecal transplants would affect gastrointestinal and autistic symptoms
- treatment group got fecal transplant, while control did not
    - collected fecal and swab microbiome data every couple weeks

---

# Background Information

- Microbiome data extraction and sequencing
    - DNA isolated
    - 16S rRNA library prep
    - barcoded primer 515f-806r
        - targets 16S V4 region
        - amplify bacterial/ archaeal 16S rRNA genes
- used 10% of data across 2 Illumina MiSeq runs
---

# Methods
![bg right:70% fit](https://github.com/qiime2/docs/raw/master/source/tutorials/images/overview.png)

---

# Methods

### Importing Data

- download fastq files using `curl` (already demultiplexed)
- remove poly-G tail and filter reads using `fastp`
- import fastq files into qiime readable format using `qiime tools import`
- trim primers using `qiime cutadapt trim-single`
<br />
- do once per run (2 runs total)

---

<style>
    background-image-padding: 1px;
</style>

# Methods

### Denoising Prep

- `qiime demux summarize` used to determine best way to denoise
    - left 13, length 150

![bg right:60% fit](plots/demux-summary-1.png)

---

# Methods

### Denoising

- 'qiime 

