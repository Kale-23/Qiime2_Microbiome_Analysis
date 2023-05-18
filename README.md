# gen711_final

<details open> <summary><H2> Background </H2></summary>

Data taken from [this](https://doi.org/10.1186/s40168-016-0225-7) Human microbiome study was used to perform a bioinformatic pathway analysis based on [this](https://docs.qiime2.org/2022.2/tutorials/fmt/) tutorial by qiime2. Overall, my goal during this process was to learn how to take raw fastq reads and turn them into results that can be shared through analysis and figures generated using bioinformatic techniques.

 <i> all code used for this project can be found under the code tabs in the method sections (I did not include an overall script.sh file because it is messy and I cannot find anything when I need to redo steps) </i>

</details>

[//]: # "End Background"

<details> <summary><H2> Methods </H2></summary>

<details> <summary><H3> Downloading Data </H3></summary>

Tools Used
- [fastp](https://github.com/OpenGene/fastp)
   - useful tool that does preprocessing on fastq files
   - filters reads (low quality, high # missing, too short, too long)
   - reports read data (qualities, kmer counts, base proportions, overrepresented sequences, GC content)
- import
   - imports fastq files into qiime readable .qza format
- cutadapt
   - used to trim off primers and adapters from the sequences
- demux summarize
   - gives reads per sample, and an average read quality plot
      - helps determine cutoff for denoising step to get rid of bad parts of reads

<details> <summary><i> code </i></summary>

- This is for the 10% subsample data from [the qiime2 FMT tutorial](https://docs.qiime2.org/2022.2/tutorials/fmt/) and is already in qza format:

``` bash
#download first and second set of qiime imported qza reads
mkdir sequences
curl -sL \
    "https://data.qiime2.org/2022.2/tutorials/fmt/fmt-tutorial-demux-1-10p.qza" > \
    "sequences/fmt-tutorial-demux-1.qza"

curl -sL \
    "https://data.qiime2.org/2022.2/tutorials/fmt/fmt-tutorial-demux-2-10p.qza" > \
    "sequences/fmt-tutorial-demux-2.qza"

#human readable output summary of sequences given to determine best way to denoise
qiime demux summarize \
  --i-data sequences/fmt-tutorial-demux-1.qza \
  --o-visualization sequences/demux-summary-1.qzv

qiime demux summarize \
  --i-data sequences/fmt-tutorial-demux-2.qza \
  --o-visualization sequences/demux-summary-2.qzv
```

- Steps below are for if the raw fastq files were used:

``` bash
#download the fastp command for single end reads
cp /tmp/gen711_project_data/fastp-single.sh .

#download first and second set of fastq files
mkdir rawFastqs
mkdir rawFastqs/1
mkdir rawFastqs/2
cp /tmp/gen711_project_data/FMT_3/fmt-tutorial-demux-1/* rawFastqs/1 
cp /tmp/gen711_project_data/FMT_3/fmt-tutorial-demux-2/* rawFastqs/2

#trim tails of reads with fastq command
mkdir trimmedFastqs
mkdir trimmedFastqs/1
mkdir trimmedFastqs/2
./fastp-single.sh 120 rawFastqs/1 trimmedFastqs1
./fastp-single.sh 120 rawFastqs/2 trimmedFastqs2

#import sequence metadata
mkdir metadata/
curl -sL \
  "https://data.qiime2.org/2022.2/tutorials/fmt/sample_metadata.tsv" > \
  "metadata/sample-metadata.tsv"

#import both sets into qiime readable qza format
mkdir sequences
qiime tools import \
   --type "SampleData[SequencesWithQuality]" \
   --input-format CasavaOneEightSingleLanePerSampleDirFmt \
   --input-path trimmed_fastqs1 \
   --output-path sequences/fmt-tutorial-demux-1.qza 

qiime tools import \
   --type "SampleData[SequencesWithQuality]" \
   --input-format CasavaOneEightSingleLanePerSampleDirFmt \
   --input-path trimmed_fastqs2 \
   --output-path sequences/fmt-tutorial-demux-2.qza

#trim primers and adapters for each sequence
qiime cutadapt trim-single \
   --i-demultiplexed-sequences sequences/fmt-tutorial-demux-1.qza  \
   --p-front TACGTATGGTGCA \
   --p-discard-untrimmed \
   --p-match-adapter-wildcards \
   --verbose \
   --o-trimmed-sequences sequences/fmt-tutorial-demux-1_trimmed.qza

qiime cutadapt trim-single \
   --i-demultiplexed-sequences sequences/fmt-tutorial-demux-2.qza \
   --p-front TACGTATGGTGCA \
   --p-discard-untrimmed \
   --p-match-adapter-wildcards \
   --verbose \
   --o-trimmed-sequences sequences/fmt-tutorial-demux-2_trimmed.qza
```
</details></details>

<details>

<summary><H3> Denoising </H3></summary>

Tools Used
- dada2 denoise-single
   - denoises the sequences
      - removes sequencing errors, artifacts, denoising, and signal enhancement
- metadata tabulate
   - outputs table of stats, not nessesary to continue but cool statistics to know
      - which reads were filtered, how many were denoised, and how many were non-chimeric
- feature-table tabulate-seqs
   - outputs table of all features and their sequences and lengths

<details> <summary><i> code </i></summary>

``` bash
#DADA2 used for denoising
mkdir repSequences
qiime dada2 denoise-single \
  --p-trim-left 13 \
  --p-trunc-len 150 \
  --p-n-threads 4 \
  --i-demultiplexed-seqs sequences/fmt-tutorial-demux-1.qza \
  --o-representative-sequences repSequences/rep-seqs-1.qza \
  --o-table repSequences/table-1.qza \
  --o-denoising-stats repSequences/stats-1.qza

qiime dada2 denoise-single \
  --p-trim-left 13 \
  --p-trunc-len 150 \
  --p-n-threads 4 \
  --i-demultiplexed-seqs sequences/fmt-tutorial-demux-2.qza \
  --o-representative-sequences repSequences/rep-seqs-2.qza \
  --o-table repSequences/table-2.qza \
  --o-denoising-stats repSequences/stats-2.qza

#visualizing denoising stats
qiime metadata tabulate \
  --m-input-file repSequences/stats-1.qza \
  --o-visualization repSequences/denoising-stats-1.qzv

qiime metadata tabulate \
  --m-input-file repSequences/stats-2.qza \
  --o-visualization repSequences/denoising-stats-2.qzv

#showing feature sequences and length
qiime feature-table tabulate-seqs \
        --i-data repSequences/rep-seqs-1.qza \
        --o-visualization repSequences/rep-seqs-1.qzv

qiime feature-table tabulate-seqs \
        --i-data repSequences/rep-seqs-2.qza \
        --o-visualization repSequences/rep-seqs-2.qzv
```
</details></details>

<details> <summary><H3> Merging Data </H3></summary>

Tools Used
- feature-table merge
   - merges the two feature tables into one
- feature-table merge-seqs
   - merges the two sets of sequences into one
- feature-table summarize
   - creates a summary table and figures about frequencies of genetic varients
- feature-table tabulate-seqs
   - merged table showing each feature's sequence an its length

<details> <summary><i> code </i></summary>

``` bash
mkdir mergedRepSequences
#merging feature table for future use
qiime feature-table merge \
  --i-tables repSequences/table-1.qza \
  --i-tables repSequences/table-2.qza \
  --o-merged-table mergedRepSequences/table.qza

#merging sequences for future use
qiime feature-table merge-seqs \
  --i-data repSequences/rep-seqs-1.qza \
  --i-data repSequences/rep-seqs-2.qza \
  --o-merged-data mergedRepSequences/rep-seqs.qza

#visualization of merged feature table
qiime feature-table summarize \
  --i-table mergedRepSequences/table.qza \
  --o-visualization mergedRepSequences/table.qzv \
  --m-sample-metadata-file metadata/sample-metadata.tsv

#visualization of merged rep sequence summary stats
qiime feature-table tabulate-seqs \
  --i-data mergedRepSequences/rep-seqs.qza \
  --o-visualization mergedRepSequences/rep-seqs.qzv
```
</details></details>

<details> <summary><H3> Alignment </H3></summary>

Tools Used:
- alignment mafft
   - aligns the features in the feature table
- alignment mask
   - removes highly variable positions that can add unwanted noise

<details> <summary><i> code </i></summary>

```bash
#creates aligned sequences
mkdir alignedSequences
qiime alignment mafft \
   --i-sequences mergedRepSequences/rep-seqs.qza \
   --o-alignment alignedSequences/aligned-sequences.qza

#denoises aligned sequences
qiime alignment mask \
--i-alignment alignedSequences/aligned-sequences.qza \
--o-masked-alignment alignedSequences/masked-aligned-sequences.qza

```

</details></details>

<details> <summary><H3> Taxanomic Assignment </H3></summary>

Tools Used:
- feature-classifier classify-sklearn
   - uses model to assign taxonomic information to the input rep sequences
   - classifier from [here](https://zenodo.org/record/6395539#.ZGE7pHbMJhE)
- metadata tabulate
   - outputs visualization showing featureID with associated taxon and confidence
- taxa barplot
   - outputs barplot visualization of taxanomic


<details> <summary><i> code </i></summary>

``` bash
#download pre-trained V4 16s human stool classifier (idk if this is the best one but I found it lol)
mkdir taxonomy
curl -s \
   "https://zenodo.org/record/6395539/files/515f-806r-human-stool-classifier.qza?download=1" > \
   "taxonomy/classifier.qza"

#outputs taxanomic information about rep sequences
qiime feature-classifier classify-sklearn \
  --i-classifier taxonomy/classifier.qza \
  --i-reads mergedRepSequences/rep-seqs.qza \
  --o-classification taxonomy/taxonomy.qza

#visualization of taxonomic output
qiime metadata tabulate \
  --m-input-file taxonomy/taxonomy.qza \
  --o-visualization taxonomy/taxonomy.qzv

#barplot of taxonomic frequencies
qiime taxa barplot \
  --i-table mergedRepSequences/table.qza \
  --i-taxonomy taxonomy/taxonomy.qza \
  --m-metadata-file metadata/sample-metadata.tsv \
  --o-visualization taxonomy/taxaBarPlot.qzv
```
</details></details>

<details> <summary><H3> Tree Creation </H3></summary>

Tools Used:
   - phylogeny fasttree
      - uses fasttree to make an unrooted tree with the aligned sequences
   - phylogeny midpoint-root
      - uses fasttree to make rooted tree
   - empress tree-plot
      - combines taxonomic information with tree
   - empress community-plot
      - combines taxonomy


<details> <summary><i> code </i></summary>

``` bash
#creating unrooted tree
mkdir tree
qiime phylogeny fasttree \
   --i-alignment alignedSequences/masked-aligned-sequences.qza \
   --o-tree tree/unrooted-tree.qza

#creating rooted tree 
qiime phylogeny midpoint-root \
   --i-tree tree/unrooted-tree.qza \
   --o-rooted-tree tree/rooted-tree.qza

#if empress is not installed (learning how to actually use conda and installing qiime2/empress is a pain in the ass, please get empress onto ron)
pip install empress
qiime dev refresh-cache

#adds taxonomic data to tree
qiime empress tree-plot \
   --i-tree tree/rooted-tree.qza \
   --m-feature-metadata-file taxonomy/taxonomy.qza \
   --o-visualization tree/empress-tree-tax.qzv

#adds metadata and taxonomic data to tree
qiime empress community-plot \
   --i-tree tree/rooted-tree.qza \
   --i-feature-table mergedRepSequences/table.qza \
   --m-sample-metadata-file metadata/sample-metadata.tsv \
   --m-feature-metadata-file taxonomy/taxonomy.qza \
   --o-visualization tree/empress-tree-tax-table.qzv
```

</details></details>

<details> <summary><H3> Diversity Analysis </H3></summary>

Tools Used:
   - feature-table filter-samples
      - filters samples based on input
      - here is used to get rid of donors because we just want to compare treatment vs control
   - diversity core-metrics-phylogenetic
      - produces alpha and beta diversity metrics and visualizations
      - sampling depth determined from data2 table output
   - diversity alpha-rarefaction
      - plots diversity against sequence depth
      - can be used to determine if selected sequence depth in previous command was enough to keep most species diversity within the samples
   - diversity alpha-group-significance
      - plots the diversity metric against variables in the metadata
   - longitudinal linear-mixed-effects
      - linear model mixed effect of two selected variables
      - used here to compare treatment + week vs alpha diveristy
   - diversity umap
      - reduces dimensions in the provided beta diversity method input
      - used here with unweighted/weighted unifrac
   - emperor plot
      - scatterplot with metadata
      - used to plot umap results of unifrac beta method
   - R
      - used to remove metadata columns with missing data
         - dplyr to filter using select(-c())
      - probably could use feature-table filter-samples but I don't know SQL
   - taxa collapse
      - adds taxonomic data to feature table
   - feature-table relative-frequency
      - converts values in feature table to relative frequencies
   - longitudinal volatility
      - creates longitudinal plot with provided diversity metrics, metadata, relative frequencies, and taxonomic information

<details> <summary><i> code </i></summary>

``` bash
#remove donor by selecting for control and treatment
#I think "[treatment-group] NOT IN ('donor')" works but I don't know SQL
qiime feature-table filter-samples \
   --i-table mergedRepSequences/table.qza \
   --m-metadata-file metadata/sample-metadata.tsv \
   --p-where "[treatment-group] IN ('control', 'treatment')" \
   --o-filtered-table mergedRepSequences/no-donor-table.qza

#produce alpha and beta diversity metrics and visualizations
#output-dir is created during run
qiime diversity core-metrics-phylogenetic \
  --i-phylogeny tree/rooted-tree.qza \
  --i-table mergedRepSequences/no-donor-table.qza \
  --p-sampling-depth 876 \
  --m-metadata-file metadata/sample-metadata.tsv  \
  --p-n-jobs-or-threads 5 \
  --output-dir core-metrics

#determine if depth is good enough to continue
qiime diversity alpha-rarefaction \
    --i-table mergedRepSequences/clean-no-donor-table.qza \
    --i-phylogeny tree/rooted-tree.qza \
    --p-max-depth 876 \
    --m-metadata-file metadata/sample-metadata.tsv \
    --o-visualization core-metrics/rarefaction.qzv \
    --verbose

#creates alpha diversity boxplots using observed features
mkdir alphaDiversity
qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics/observed_features_vector.qza \
  --m-metadata-file metadata/sample-metadata.tsv \
  --o-visualization alphaDiversity/alpha-group-sig-obs-feats.qzv

#creates alpha diversity linear mixed effects plot and summary
qiime longitudinal linear-mixed-effects \
  --m-metadata-file metadata/sample-metadata.tsv core-metrics/observed_features_vector.qza \
  --p-state-column week \
  --p-group-columns treatment-group \
  --p-individual-id-column subject-id \
  --p-metric observed_features \
  --o-visualization alphaDiversity/week-treatmentVScontrol.qzv

#reduces the dimensions of the unifrac beta diverisity metric
qiime diversity umap \
  --i-distance-matrix core-metrics/unweighted_unifrac_distance_matrix.qza \
  --o-umap core-metrics/uu-umap.qza

qiime diversity umap \
  --i-distance-matrix core-metrics/weighted_unifrac_distance_matrix.qza \
  --o-umap core-metrics/wu-umap.qza

#plots the umap
qiime emperor plot \
  --i-pcoa core-metrics/uu-umap.qza \
  --m-metadata-file metadata/sample-metadata.tsv core-metrics/uu-umap.qza core-metrics/faith_pd_vector.qza core-metrics/evenness_vector.qza core-metrics/shannon_vector.qza \
  --o-visualization core-metrics/uu-umap-emperor.qzv

qiime emperor plot \
  --i-pcoa core-metrics/wu-umap.qza \
  --m-metadata-file metadata/sample-metadata.tsv core-metrics/uu-umap.qza core-metrics/faith_pd_vector.qza core-metrics/evenness_vector.qza core-metrics/shannon_vector.qza \
  --o-visualization core-metrics/wu-umap-emperor.qzv
```

-Using R to remove NaN values so the next step will work
``` R
#readr for tsv impor/export, dplyr for filtering
library(readr)
library(dplyr)

df <- read_tsv("metadata/sample-metadata.tsv")

#these three rows have missing values that cause qiime2 viewer to not work in the longitudinal plot
df2 <- df %>%
   select(-c("gsrs", "gsrs-diff", "administration-route"))

#somehow these together remove the Na values R puts in tsv outputs
df2[is.na(df2)] <- ""
write_tsv(df2, "metadata/clean-metadata.tsv", na = "")
```

``` bash
#refilter table with the new metadata file
qiime feature-table filter-samples \
   --i-table mergedRepSequences/table.qza \
   --m-metadata-file metadata/clean-metadata.tsv \
   --p-where "[treatment-group] IN ('control', 'treatment')" \
   --o-filtered-table mergedRepSequences/clean-no-donor-table.qza

#add taxinomic data to table
qiime taxa collapse \
  --i-table mergedRepSequences/clean-no-donor-table.qza \
  --i-taxonomy taxonomy/taxonomy.qza \
  --p-level 6 \
  --o-collapsed-table mergedRepSequences/clean-no-donor-genus-table.qza

#converts counts to relative frequencies
qiime feature-table relative-frequency \
  --i-table mergedRepSequences/clean-no-donor-genus-table.qza \
  --o-relative-frequency-table mergedRepSequences/clean-no-donor-genus-relFreq-table.qza

#creates longitudinal plot visualization
mkdir longitudinal
qiime longitudinal volatility \
  --i-table mergedRepSequences/clean-no-donor-genus-relFreq-table.qza \
  --p-state-column week \
  --m-metadata-file metadata/clean-metadata.tsv core-metrics/uu-umap.qza core-metrics/faith_pd_vector.qza core-metrics/evenness_vector.qza core-metrics/shannon_vector.qza \
  --p-individual-id-column subject-id \
  --p-default-group-column treatment-group \
  --o-visualization longitudinal/volatility-plot.qzv
```

</details></details>

</details>

[//]: # "End Methods"

<details> <summary><H2> Results </H2></summary>

stuff

</details>

[//]: # "End Results"

<details> <summary><H2> Helpful Links and Bibliography </H2></summary>

- project is based off of the [qiime2 FMT tutorial](https://docs.qiime2.org/2022.2/tutorials/fmt/)

- [original paper](https://microbiomejournal.biomedcentral.com/articles/10.1186/s40168-016-0225-7#Sec25) where data is from

- [qiime2 overview](https://github.com/qiime2/docs/blob/master/source/tutorials/overview.rst) in github provides some nice visualizations about the bioinformatic pathway it provides

<details> <summary> example visualization </summary>

![example of picture showing bioinformatic pathway](https://github.com/qiime2/docs/raw/master/source/tutorials/images/overview.png)

</details>

- [Silva classifier list](https://zenodo.org/record/6395539#.ZGE7pHbMJhE)

- helpful [example workflow](https://bioinformaticsworkbook.org/dataAnalysis/Metagenomics/Qiime2.html#gsc.tab=0) of similar study in qiime2

- helpful [youtube playlist](https://www.youtube.com/playlist?list=PLbVDKwGpb3XmkQmoBy1wh3QfWlWdn_pTT) from the qiime2 youtube channel

- [chatGPT3](https://chat.openai.com/) helps explain what all the outputs mean and what can be done wtih them instead of searching through qiime2 documentation
   - often wrong about stuff so double check what it says

</details>

[//]: # "End Helpful Links and Bibliography"