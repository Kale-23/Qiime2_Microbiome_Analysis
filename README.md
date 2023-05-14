# gen711_final

<details open> <summary><H2> Background </H2></summary>

Data taken from [this](https://doi.org/10.1186/s40168-016-0225-7) Human microbiome study was used to perform a bioinformatic pathway analysis based on [this](https://docs.qiime2.org/2022.2/tutorials/fmt/) tutorial by qiime2. Overall, my goal during this process was to learn how to take raw fastq reads and turn them into results that can be shared through analysis and figures generated using bioinformatic techniques. To start, my goal was to see the overall phylogenetic relationship of all included microbiota by generating a phylogenetic tree. 

 Later in the analysis, I began to see more questions that I could ask about the given data (such as [this](https://forum.qiime2.org/t/tutorial-integrating-qiime2-and-r-for-data-visualization-and-analysis-using-qiime2r/4121) tutorial on PCA analysis of microbiome data), and I experimented with tools in qiime2's library and beyond that could potentially answer these questions. Most of these tools I have not had the opprotunity to finish learning before the project is due, but I plan to continue learning how to use them for future projects.

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

- If I were to start with the data given in lab, the following code would be run:

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

<details> <summary><i> code </i></summary>

``` bash
#DADA used for denoising
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


</details>

[//]: # "End Methods"

<details> <summary><H2> Results </H2></summary>

stuff

</details>

[//]: # "End Results"