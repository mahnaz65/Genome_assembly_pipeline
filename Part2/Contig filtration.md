In this part, we should investigate three types of contigs to exclude: 
Contaminants: Blobtools
Haplotigs: purge_haplotigs
Contigs from chloroplast or mitochondria: BLAST


### Blobtools
Three file prep steps, then GUI:

1. blast (Diamond or mmseqs2) hit for each contig/scaffold.
2. get deep coverage file (any deep bam).
3. get BUSCOs
The pipeline documented [here](https://blobtools.readme.io/docs/what-is-blobtools)
### purge_haplotigs
step 1:
Generate a coverage histogram by running the first script. This script will produce a histogram png image file for you to look at and a BEDTools 'genomecov'-like file that you'll need for STEP 2. 

`$ purge_haplotigs hist -b <bamprefix>.bam -g <prefix>.fa -t 16`
step 2:
Run the second script using the cutoffs from the previous step to analyse the coverage on a contig by contig basis. This script produces a contig coverage stats csv file with suspect contigs flagged for further analysis or removal. 

`$ purge_haplotigs cov -i <bamprefix>.bam.200.gencov -l 15 -m 75 -h 190 -o coverage_stats.csv`

Step 3:
Run the purging pipeline. This script will automatically run a BEDTools windowed coverage analysis (if generating dotplots), and minimap2 alignments to assess which contigs to reassign and which to keep. The pipeline will make several iterations of purging. Optionally, parse repeats -r in BED format for improved handling of repetitive regions. 

`$ purge_haplotigs purge -d -g <prefix>.fa -c coverage_stats.csv -b <bamprefix>.bam -t 16` 

We will have five files
```
<prefix>.fasta: These are the curated primary contigs
<prefix>.haplotigs.fasta: These are all the haplotigs identified in the initial input assembly.
<prefix>.artefacts.fasta: These are the very low/high coverage contigs (identified in STEP 2). NOTE: you'll probably have mitochondrial/chloroplast/etc. contigs in here with the assembly junk.
<prefix>.reassignments.tsv: These are all the reassignments that were made, as well as the suspect contigs that weren't reassigned.
<prefix>.contig_associations.log: This shows the contig "associations" e.g
```
