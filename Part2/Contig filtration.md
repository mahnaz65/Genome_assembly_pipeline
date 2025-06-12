In this part, we should investigate three types of contigs to exclude: 
Contaminants: Blobtools
Haplotigs: purge_haplotigs
Contigs from chloroplast or mitochondria: BLAST


### [Blobtools](https://blobtools.readme.io/docs/what-is-blobtools) 

Three file prep steps, then GUI:

1. blast (Diamond or mmseqs2) hit for each contig/scaffold.
2. get deep coverage file (any deep bam).
3. get BUSCOs
The pipeline documented [here](https://blobtools.readme.io/docs/what-is-blobtools)
### [Purge_haplotigs](https://bitbucket.org/mroachawri/purge_haplotigs/src/master/) 

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
### BLAST to Refseq
 We identified and removed non-nuclear DNA by blasting the primary assembly against a reference plastid sequence database from RefSeq using MMseqs2  to detect and filter plastid DNA, including mitochondrial and chloroplast sequences.
Step 1:
Create a database from your concatenated sequences (Sequences from chloroplast and mitochondria) 

`$mmseqs createdb concatenetedSeq.fa concatenetedSeqDB` 

Step2:
Create a database for the assembly 

`$mmseqs createdb <prefix>.fa <prefix>db` 

Step3:
Search the database 

`$mmseqs search -a --start-sens 1 --sens-steps 3 -s 7 --search-type 3 --threads 16 <prefix>db concatenetedSeqDB blast_results $SCRATCHDIR` 

Step4:
Convert mmseq2 results to BLAST fromat (multiple cores, submit as batch) 

`$mmseqs convertalis --search-type 3 --threads 16 <prefix>db concatenetedSeqDB blast_results blast_results.m8` 

Step5:
Keep best hits only 

`$sort -k 1,1 -k 12,12rn -S 8G --parallel 8 blast_results.m8 | sort -u -k 1,1 -S 8G --parallel 8 > blast_results.best.hits.only.m8` 

Step6:
Get description of accession numbers of best hits 

`$cut -f 2 blast_results.best.hits.only.m8 | sort -u > blast_results.best.hits.only.accession.number.out` 

Step7:
Run a python script 

`$python3 $BIN/get_definition_from_genbank_ids.py -i blast_results.best.hits.only.accession.number.out -o blast_results.best.hits.only.accession.number.and.definitions.out`
