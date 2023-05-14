# gen711_final

<details> <summary><H2> Background </H2></summary>

</details>


<details> <summary><H2> Methods </H2></summary>

<details> <summary><H3> Downloading Data </H3></summary>

Tools Used
- import
- cutadapt
- demux summarize

<details> <summary><i> code </i></summary>

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
- This is for the 10% subsample data from [the qiime2 FMT tutorial](https://docs.qiime2.org/2022.2/tutorials/fmt/) and is already in qza format
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
qiime tools import \
mkdir sequences
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
</details>
</details>

<details>

<summary><H3> Denoising </H3></summary>

Tools Used
- denoise-single
- metadata tabulate

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
</details>
</details>

<details> <summary><H3> Merging Data </H3></summary>

Tools Used
- feature-table merge
- feature-table merge-seqs
- feature-table summarize

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
</details>
</details>

</details>
</details>