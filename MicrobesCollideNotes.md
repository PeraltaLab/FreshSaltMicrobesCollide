# Microbes Collide Project Setup

## Install QIIME (Mac; prereq: Anaconda3)

    wget https://data.qiime2.org/distro/core/qiime2-2019.7-py36-osx-conda.yml
    conda env create -n qiime2-2019.7 --file qiime2-2019.7-py36-osx-conda.yml

    # OPTIONAL CLEANUP
    rm qiime2-2019.7-py36-osx-conda.yml

conda activate qiime2-2019.7

# To deactivate an environment, run:
# conda deactivate

CSI Data: https://github.com/PeraltaLab/CSI_Dispersal
https://www.ncbi.nlm.nih.gov/bioproject/PRJNA615001
Use link to navigate to the bio project and the new SRA download page (SRA Run Selector Tool)
Get two files: 
CSI_SRR_Acc_List.txt (Accession List, rename to add CSI_)
CSI_SraRunTable.txt (Metadata, rename to add CSI_)

SRA Toolkit Instructions: https://github.com/ncbi/sra-tools/wiki/02.-Installing-SRA-Toolkit

On Mac:
Brew install sratoolkit

On Ubuntu:
wget --output-document sratoolkit.tar.gz http://ftp-trace.ncbi.nlm.nih.gov/sra/sdk/current/sratoolkit.current-ubuntu64.tar.gz
tar -vxzf sratoolkit.tar.gz
export PATH=$PATH:$PWD/sratoolkit.2.10.7-ubuntu64/bin

configure SRA Toolkit
# You don't need to do this
# vdb-config -i # Reset the download folder (use external since it is large)

Download the data:
while read p; do
  echo "$p"
  prefetch $p
done < CSI_SRR_Acc_List.txt

while read p; do
  echo "$p"
  fasterq-dump $p
done < CSI_SRR_Acc_List.txt 

mkdir raw
mkdir analysis
mv ./*.fastq ./raw

Using CSI_SraRunTable.txt create manifest (this must be done manually)
must use absolute path 
e.g. $PWD

sample-id	forward-absolute-filepath	 reverse-absolute-filepath	
ECU_CSI_102	$PWD/raw/SRR11411543_1.fastq	$PWD/raw/SRR11411543_2.fastq	

Save as CSI_manifest.txt

# Import Data
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path CSI_manifest.txt \
  --output-path demux-CSI.qza \
  --input-format PairedEndFastqManifestPhred33V2

### Quality Check
qiime demux summarize \
 --i-data demux-CSI.qza \
 --o-visualization demux-CSI.qzv

# check for Qiime2 view for quality and verify length
qiime tools view demux-CSI.qzv

##Trimming forward and reverse primers to keep only the V4 region
qiime cutadapt trim-paired \
  --i-demultiplexed-sequences demux-CSI.qza \
  --p-front-f 'GTGYCAGCMGCCGCGGTAA' \
  --p-front-r 'GGACTACHVGGGTWTCTAAT' \
  --o-trimmed-sequences demux-CSI-trimmed-V4.qza \
  --verbose
  
#########
### DADA2 to identify ESVs. We ran this analysis on each independent study separately and merged the ESV tables and representative sequences afterwards (see merging steps below in Qiime 2)
####Processed on R version 3.5.1 (2018-07-02) 
####DADA2: 1.10.0 / Rcpp: 1.0.2 / RcppParallel: 4.4.3 
qiime dada2 denoise-paired \
  --i-demultiplexed-seqs demux-CSI-trimmed-V4.qza \
  --p-trim-left-f 0 \
  --p-trunc-len-f 200 \
  --p-trim-left-r 0 \
  --p-trunc-len-r 200 \
  --p-n-threads 0 \
  --o-table CSI_table.qza \
  --o-representative-sequences CSI_repseqs.qza \
  --o-denoising-stats CSI_denoisestats.qza \
  --verbose

qiime metadata tabulate \
  --m-input-file CSI_denoisestats.qza \
  --o-visualization CSI_stats-dada2.qzv

qiime tools view CSI_stats-dada2.qzv
# look for any substantial drops

qiime feature-table summarize \
  --i-table CSI_table.qza \
  --o-visualization CSI_table.qzv \
  --m-sample-metadata-file CSI_sample-metadata.txt

qiime tools view CSI_table.qzv
# Verify number of samples, and sequencing depth.

qiime feature-table tabulate-seqs \
  --i-data CSI_repseqs.qza \
  --o-visualization CSI_reps.qzv

qiime tools view CSI_reps.qzv
# Sequence length, and should start with “TAC….”, and number of SVs. (edited) 

# Export Feature table
qiime tools export \
  --input-path CSI_table.qza \
  --output-path exported-feature-table

#### Done #####




Potential Bioinformatic script in Qiime2:
DADA2 version 1.10.0 / Rcpp: 1.0.2 / RcppParallel: 4.4.3 
### Import to Qiime2
# turn on Qiime2
source activate qiime2-2019.7 
# Older version, but it’s the version used by MBSP (different versions impact Dada2 output)


# Import Data
# Method 1: Manifest method (useful for NCBI/SRA downloads)
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path Author-Year_manifest.csv \
  --output-path demux-Author-Year.qza \
  --input-format SingleEndFastqManifestPhred33
# Method 2: Cassava formatted input if fastq names are: "SampleID_S50_L001_R1_001.fastq.gz". - you need to prepare a manifest csv file with 3 columns (sample-id,absolute-filepath,direction) to tell qiime2 where to find your fastq files:
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path 'Folder' \
  --input-format CasavaOneEightSingleLanePerSampleDirFmt \
  --output-path demux-Author-Year.qza
#########
### Quality Check
qiime demux summarize \
 --i-data demux-Author-Year.qza \
 --o-visualization demux-Author-Year.qzv
qiime tools view demux-Author-Year.qzv
# check for Qiime2 view for quality and verify length
##Trimming forward and reverse primers to keep only the V4 region
cutadapt trim-paired
–i-demultiplexed-sequences  demux-Author-Year.qza
–p-front-f GTGYCAGCMGCCGCGGTAA
–p-front-r GGACTACHVGGGTWTCTAAT
–verbose
–o-trimmed-sequences demux-Author-Year-trimmed-V4.qza
#########
### DADA2 to identify ESVs. We ran this analysis on each independent study separately and merged the ESV tables and representative sequences afterwards (see merging steps below in Qiime 2)
####Processed on R version 3.5.1 (2018-07-02) 
####DADA2: 1.10.0 / Rcpp: 1.0.2 / RcppParallel: 4.4.3 
qiime dada2 denoise-paired \
  --i-demultiplexed-seqs demux-Author-Year-trimmed-V4.qza \
  --p-trim-left-f 0 \ # No trimming, cause we’ve already done this with cutadapt
  --p-trim-left-r 0 \
  --p-trunc-len-f 200 \ # Aggressive truncating, but fine since the overlap is very high in V4.
  --p-trunc-len-r 200 \
  --p-n-threads 0 \
  --o-table Author-Year_table.qza \
  --o-representative-sequences Author-Year_repseqs.qza \
  --o-denoising-stats Author-Year_denoisestats.qza \
  --verbose
qiime metadata tabulate \
  --m-input-file Author-Year_denoisestats.qza \
  --o-visualization Author-Year_stats-dada2.qzv
qiime tools view Author-Year_stats-dada2.qzv
# look for any substantial drops
qiime feature-table summarize \
  --i-table Author-Year_table.qza \
  --o-visualization Author-Year_table.qzv \
  --m-sample-metadata-file Author-Year_sample-metadata.tsv
qiime tools view Author-Year_table.qzv
# Verify number of samples, and sequencing depth.
qiime feature-table tabulate-seqs \
  --i-data Author-Year_repseqs.qza \
  --o-visualization Author-Year_reps.qzv
qiime tools view Author-Year_reps.qzv
# Sequence length, and should start with “TAC….”, and number of SVs. (edited) 