# Genome_assembly_pipeline
To learn the steps of genome assembly using Hifiasm  
Genome assembly workshop

# Raw data

I created a subsample from HiFi PacBio read that I have. With these subsamples, you should run the analyses on your local computer. So far, I do not have Hi-C data
related to this sample, so, we run the pipeline without the Hi-C data.
As you can see in the names of the samples, one of them is smaller. you can try which one that works on your local computer. Please download the files from the links below:

https://drive.google.com/file/d/16gE0bZWr3eby1jbJR1oITroP3scXxwfJ/view?usp=drive_link
https://drive.google.com/file/d/1GtP7SLnQEGgsxywnZqsOaTIDWN5QYsLs/view?usp=drive_link

### Installation

#### Anaconda
If you just install Ubuntu, perhaps you need to install Anaconda or Miniconda. I prefer Anaconda. If you installed Ubuntu, so, you should know that
Ubuntu is Debian-based, so, always you should install packages for Debian. Follow the link below to install Anaconda.
https://docs.anaconda.com/free/anaconda/install/linux/

Please read the page carefully, and do exactly what it wrote. Just copy the commands in the order, change the directory when needed, and type yes when needed. All
is explained on this page better than my explanation.

#### JAVA

This is better to be installed as some software is java-based. They might be compatible with a different version, but we installed the last version and let's see if
some software needs an older version, we can install that version afterward.
To install java follow commands below:  
```
  sudo apt-get install openjdk-18-jre
  sudo apt-get install openjdk-18-jdk openjdk-18-jre-headless
```
Generally, there are three steps to generate the assembly file. First is generating the initial assembly, then scaffolding, and then polishing.

## 1. Initial assembly:
In this step, we need Hifi read, Hi-c reads, and an estimation of the genome size from Kmer analysis. 
Steps:  
1.1 Estimation of the genome size  
1.2 Generating the initial assembly  
1.3 Quality control of initial assemblies  
1.4 Remove organellar and erroneous contigs from the initial assemblies  
1.5 Rerun the steps of Step 3 on the filtered assemblies  
1.6 Rerun the steps of Step 3 on the filtered assemblies  

### 1.1 Estimation of the genome size
Before starting a de novo genome assembly project, it is useful to estimate the expected genome size. Traditionally, DNA flow cytometry was considered the golden standard for estimating the genome size. Nowadays, experimental methods have been replaced by computational approaches. One of the widely used genome profiling methods is based on the analysis of k-mer frequencies. It allows one to provide information not only about the genomic complexity, such as the genome size and levels of heterozygosity and repeat content but also about the data quality. There are different software that can calculate k-mer frequency such as Jellyfish, KMC, KAT, Meryl, .... In this step, I first ran Jellyfish to get the hist file and then I used [findGSE](https://github.com/schneebergerlab/findGSE) to create the plot. But, I gave weird peaks, like the first peak was taller than the second one even with different k-mer sizes. So, I switched to use [KMC](https://github.com/refresh-bio/KMC) and [GenomeScope2](http://qb.cshl.edu/genomescope/genomescope2.0/). Outputs and scripts are in KMC.GenomeScope folder.  
#### KMC  
KMC is a disk-based program for counting k-mers from (possibly gzipped) FASTQ/FASTA files.  
Here is the script:  
```
#PBS -l select=1:ncpus=16:mem=70gb:scratch_local=100gb
#PBS -l walltime=2:00:00
module add kmc/3.1.1

INPUT=/storage/brno2/home/mahnaz_nezami/rawData/conAdapt/Erysimum/unzip_file
OUTPUT=/storage/brno2/home/mahnaz_nezami/outputs/KMC/tmp


cd $SCRATCHDIR
cp $INPUT/*.fastq .
ls *.fastq > Files
kmc -k21 -t16 -m64 -ci1 -cs10000 @Files reads $SCRATCHDIR
kmc_tools transform reads histogram reads.histo -cx10000
```
This script produces these outputs: *reads.histo* ,*reads.kmc_pre* ,*reads.kmc_suf*. Then we can upload histo file into [GenomeScope2](http://qb.cshl.edu/genomescope/genomescope2.0/) to create the plots. 
 
### 1.2 Generating the initial assembly  
Run hifiasm (https://github.com/chhylp123/hifiasm) to generate initial assembly.
   - If the species is (nearly) homozygous, run hifiasm with the `-l0`parameter and include the HiFi reads as input only. This disables haplotype purging, which would erroneously remove repetitive sequences (that are mistakingly seen as alternate haplotypes) otherwise. This will generate a primary assembly graph as output (`prefix.p_ctg.gfa`) 
   - If the species is heterozygous, run hifiasm with both HifI reads and Hi-C reads as input. This will generate two-phased assembly graphs, corresponding to the two haplotypes (`prefix.dip.hap1.p_ctg.gfa` and `prefix.dip.hap2.p_ctg.gfa`)
Here is the script:
```
#!/bin/bash

#PBS -l select=1:ncpus=16:mem=100gb:scratch_local=100gb
#PBS -l walltime=24:00:00
module add hifiasm/0.19.7


INPUT=/storage/brno12-cerit/home/mahnaz_nezami/rawData/conAdapt/Noccaea
OUTPUT=/storage/brno12-cerit/home/mahnaz_nezami/rawData/conAdapt/Noccaea

cd $SCRATCHDIR
mkdir out
cp $INPUT/*fastq.gz .
hifiasm -o ./out/Noccaea.asm -t 16 ./*fastq.gz
```
#### Hifiasm generates different types of assemblies based on the input data;  
In general, hifiasm generates the following assembly graphs in the GFA format:

`prefix`.r_utg.gfa: haplotype-resolved raw unitig graph. This graph keeps all haplotype information.

`prefix`.p_utg.gfa: haplotype-resolved processed unitig graph without small bubbles. Small bubbles might be caused by somatic mutations or noise in data, which are not the real haplotype information. Hifiasm automatically pops such small bubbles based on coverage. The option --hom-cov affects the result. See homozygous coverage setting for more details. In addition, the option -p forcedly pops bubbles.

`prefix`.p_ctg.gfa: assembly graph of primary contigs. This graph includes a complete assembly with long stretches of phased blocks.

`prefix`.a_ctg.gfa: assembly graph of alternate contigs. This graph consists of all contigs that are discarded in primary contig graph.

`prefix`.*hap*.p_ctg.gfa: phased contig graph. This graph keeps the phased contigs.

Hifiasm outputs *.r_utg.gfa and *.p_utg.gfa in any cases. Specifically, hifiasm outputs the following assembly graphs in trio-binning mode:

`prefix`.dip.hap1.p_ctg.gfa: fully phased paternal/haplotype1 contig graph keeping the phased paternal/haplotype1 assembly.

`prefix`.dip.hap2.p_ctg.gfa: fully phased maternal/haplotype2 contig graph keeping the phased maternal/haplotype2 assembly.

With Hi-C partition options, hifiasm outputs:

`prefix`.hic.p_ctg.gfa: assembly graph of primary contigs.

`prefix`.hic.hap1.p_ctg.gfa: fully phased contig graph of haplotype1 where each contig is fully phased.

`prefix`.hic.hap2.p_ctg.gfa: fully phased contig graph of haplotype2 where each contig is fully phased.

`prefix`.hic.a_ctg.gfa (optional with --primary): assembly graph of alternate contigs.

Hifiasm generates the following assembly graphs only with HiFi reads in default:

`prefix`.bp.p_ctg.gfa: assembly graph of primary contigs.

`prefix`.bp.hap1.p_ctg.gfa: partially phased contig graph of haplotype1.

`prefix`.bp.hap2.p_ctg.gfa: partially phased contig graph of haplotype2.

If the option --primary or -l0 is specified, hifiasm outputs:

`prefix`.p_ctg.gfa: assembly graph of primary contigs.

`prefix`.a_ctg.gfa: assembly graph of alternate contigs.
. four assembly files *.asm.bp.hap1.p_ctg.gfa*, *.asm.bp.hap2.p_ctg.gfa*, *asm.bp.p_ctg.gfa*  
    *hap1* and *hap2* can be thought to represent the two haplotypes in a diploid genome, though with occasional switch errors. The frequency of switches is determined by the heterozygosity of the input sample.  

At the first run, hifiasm saves corrected reads and overlaps to disk as NA12878.asm.*.bin. It reuses the saved results to avoid the time-consuming all-vs-all overlap calculation next time. You may specify -i to ignore precomputed overlaps and redo overlapping from raw reads. You can also dump error corrected reads in FASTA and read overlaps in PAF with  
```
hifiasm -o NA12878.asm -t 32 --write-paf --write-ec /dev/null
```
Hifiasm purges haplotig duplications by default. For inbred or homozygous genomes, you may disable purging with option -l0. Old HiFi reads may contain short adapter sequences at the ends of reads. You can specify -z20 to trim both ends of reads by 20bp. For small genomes, use -f0 to disable the initial bloom filter which takes 16GB of memory at the beginning. For genomes much larger than human, applying -f38 or even -f39 is preferred to save memory on k-mer counting.  
#### Hi-C integration
Hifiasm can generate a pair of haplotype-resolved assemblies with paired-end Hi-C reads:  
```
hifiasm -o NA12878.asm -t32 --h1 read1.fq.gz --h2 read2.fq.gz HiFi-reads.fq.gz
```
In this mode, each contig is supposed to be a haplotig, which by definition comes from one parental haplotype only. Hifiasm often puts all contigs from the same parental chromosome in one assembly. It has cleanly separated chrX and chrY for a human male dataset. Nonetheless, phasing across centromeres is challenging. Hifiasm is often able to phase entire chromosomes but it may fail in rare cases. Also, contigs from different parental chromosomes are randomly mixed as it is just not possible to phase across chromosomes with Hi-C.

Hifiasm does not perform scaffolding for now. You need to run a standalone scaffolder such as SALSA or 3D-DNA to scaffold phased haplotigs.  
For samples with high heterozygosity rate, a common issue is that one assembly is much larger than another one. To fix this issue, please set smaller value for -s (default: 0.55). Another possibility is that hifiasm misidentifies coverage threshold for homozygous reads. In this case, please set --hom-cov to homozygous coverage peak. See ["How can I tweak parameters to improve Hi-C integrated assembly?"](https://hifiasm.readthedocs.io/en/latest/faq.html#hic-iss) for more details.

At the first run, hifiasm saves the alignment of Hi-C reads to disk as *hic*.bin. It reuses the saved results to avoid Hi-C alignment next time. Please note that *hic*.bin should be deleted when tuning any parameters affecting *p_utg*gfa. Since v0.15.5, hifiasm can detect such changes and renew Hi-C bin files automatically. There are several parameters which do not change *p_utg*gfa, including -s, --seed, --n-weight, --n-perturb, --f-perturb and --l-msjoin.  
#### [This link](https://hifiasm.readthedocs.io/en/latest/faq.html#can-i-generate-hifi-only-assembly-first-and-then-add-hi-c-or-trio-data-later) is recommended to look!!! 
#### Convert the GFA files to FASTA files, using the following line of code:  
`cat insert_file_name.gfa | grep '^S' | cut -f2,3 | awk '{print ">"$1"\n"$2}' > insert_file_name.fa`  

## 1.3 Quality control of initial assemblies  
After generating the promary assembly, we should check the quality of assembly file using these three different approaches:  
   - Run QUAST (https://github.com/ablab/quast) to check the number of contigs, N50, and assembly size. Check whether assembly size is similar to genome size estimate.
   - Run BUSCO (https://busco.ezlab.org/) to check the "functional completeness" of the assembly. >96% of BUSCOs should be present, some perhaps in multiple copies if the species underwent a recent whole genome duplication. 
   - Run Merqury (https://github.com/marbl/merqury) using HiFi k-mers to check quality value (QV) and genome completeness.
   - Resolve any issues that appear from the above quality control steps.
## 1.3.1 QUAST  
QUAST stands for QUality ASsessment Tool. It evaluates genome/metagenome assemblies by computing various metrics. The QUAST package works both with and without reference genomes. However, it is much more informative if at least a close reference genome is provided along with the assemblies. The tool accepts multiple assemblies and thus is suitable for comparison.  
Here is the script:  
```
#!/bin/bash

#PBS -l select=1:ncpus=16:mem=100gb:scratch_local=100gb
#PBS -l walltime=100:00:00

module add quast/4.6.3


INPUT=/storage/brno2/home/mahnaz_nezami/outputs/hifiasm/conAdapt/Erysimum/hetero_hifiRun
OUTPUT=/storage/brno2/home/mahnaz_nezami/outputs/quast/conAdapt/Erysimum/quast_results_primaryAsm

cd $SCRATCHDIR
mkdir out
cp $INPUT/Erysimum.asm.bp.p_ctg.fa .
python /software/quast/4.6.3/quast.py ./Erysimum.asm.bp.p_ctg.fa
```
These are the outputs:  
```
report.txt      summary table
report.tsv      tab-separated version, for parsing, or for spreadsheets (Google Docs, Excel, etc)  
report.tex      Latex version
report.pdf      PDF version, includes all tables and plots for some statistics
report.html     everything in an interactive HTML file
icarus.html     Icarus main menu with links to interactive viewers
contigs_reports/        [only if a reference genome is provided]
  misassemblies_report  detailed report on misassemblies
  unaligned_report      detailed report on unaligned and partially unaligned contigs
k_mer_stats/            [only if --k-mer-stats is specified]
  kmers_report          detailed report on k-mer-based metrics
reads_stats/            [only if reads are provided]
  reads_report          detailed report on mapped reads statistics
```
## 1.3.2 BUSCO  
BUSCO completeness assessment employs sets of Benchmarking Universal Single-Copy Orthologs from OrthoDB (www.orthodb.org) to provide quantitative measures of the completeness of genome assemblies, annotated gene sets, and transcriptomes in terms of expected gene content. Genes that make up the BUSCO sets for each major lineage are selected from orthologous groups with genes present as single-copy orthologs in at least 90% of the species. While allowing for rare gene duplications or losses, this establishes an evolutionarily informed expectation that these genes should be found as single-copy orthologs in the genome of any newly-sequenced species.  
Choose and download a library of lineage-specific BUSCO data (http://busco.ezlab.org), we recommend using the largest library possible for your species.  
```
#!/bin/bash

#PBS -l select=1:ncpus=32:mem=100gb:scratch_local=100gb
#PBS -l walltime=50:00:00

module add conda-modules/py37
conda activate busco_v5.4.3_py3.8


INPUT=/storage/brno2/home/mahnaz_nezami/outputs/hifiasm/conAdapt/Erysimum/hetero_hifiRun
OUTPUT=/storage/brno2/home/mahnaz_nezami/outputs/busco/conAdapt/Erysimum/primary_assembly_file/brassi_lineage

cd $SCRATCHDIR

cp -r $INPUT/Erysimum.asm.bp.p_ctg.fa .
busco -m genome --in Erysimum.asm.bp.p_ctg.fa -o out_busco_brassica -l brassicales_odb10
```
Successful execution of the BUSCO assessment pipeline will create a directory named name_OUTPUT where ‘name’ is your assigned name for the assessment run. The directory will contain several files and directories:

Files
short_summary_ Contains summary results in BUSCO notation and a brief breakdown of the metrics

full_table_ Complete results in tabular format with coordinates, scores and lengths of BUSCO matches

training_set_ Set of complete BUSCO matches used for training Augustus Only created during genome assessment

_tblastn Results in tabular format of tBLASTn searches with BUSCO consensus sequences

Directories
augustus_ Augustus-predicted genes Only created during genome assessment

augutus_proteins Corresponding Augustus-predicted proteins Only created during genome assessment

Selected Complete BUSCO matches, used for training Augustus

gb Complete BUSCO matches, GenBank format

gffs Complete BUSCO matches, GFF format

hmmer_output Tabular format HMMER output of searches with BUSCO HMMs  

