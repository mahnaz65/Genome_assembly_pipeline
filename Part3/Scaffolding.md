# Scaffolding workflow Brassicaceae

### Input

* A FASTA file containing contigs (referred to as contigs.fa here), including its index file (referred to as contigs.fa.fai here, can be generated with samtools faidx)
* BAM file of Hi-C short-read data aligned to the contigs (referred to as alignments.bam here)
* BED file of Hi-C short-read data aligned to the contigs (referred to as alignments.bed here)

### Dependencies

* SALSA2 (https://github.com/marbl/SALSA), for getting the run_pipeline.py script, referred to as SALSA/run_pipeline.py here
* YAHS (https://github.com/c-zhou/yahs), for getting the juicer binary, referred to as yahs/juicer here
* Juicertools version 1.9.0 (https://github.com/aidenlab/juicer/wiki/Download) for getting the juicer_tools.1.9.9_jcuda.0.8.jar file, referred to as juicer_dir/scripts/common/juicer_tools.1.9.9_jcuda.0.8.jar here
* Juicebox Assembly Tools (JBAT), installed on your local computer. Get version 1.11.08 through `wget https://s3.amazonaws.com/hicfiles.tc4ga.com/public/Juicebox/Juicebox_1.11.08.jar`. It is old, but the version that works for me without issues. 



### Preparing input files for JBAT

Run SALSA2 to generate scaffolds, making sure that the AGP file uses coordinates of the original contigs (not the renamed ones generated after SALSA2 corrects misassemblies). Change -e and -s according to the used restriction enzymes and the estimated genome size of the assembly:

* SALSA/run_pipeline.py -a contigs.fa -l contigs.fa.fai \
  -b alignments.bed -e <restriction_enzymes> -o scaffolds_original_coordinates -O -m yes -i 10 -s <estimated_genome_size>

Get into the directory containing the scaffolds (scaffolds_original_coordinates) and produce a .hic and .assembly file that will be used as input for Juicebox Assembly Tools. We will use the procedure as described at https://github.com/c-zhou/yahs in the "Manual curation with Juicebox (JBAT)" section: 

* yahs/juicer pre -a -o SALSA2_scaffolds_out_JBAT alignments.bam scaffolds_FINAL.original-coordinates.agp contigs.fa.fai >SALSA2_scaffolds_out_JBAT.log 2>&1

One of the output files of this step is SALSA2_scaffolds_out_JBAT.log, which will have a line at the end telling you how to run juicer_tools to generate the final Hi-C file. For instance:

* java -jar -Xmx36G juicer_dir/scripts/common/juicer_tools.1.9.9_jcuda.0.8.jar pre SALSA2_scaffolds_out_JBAT.txt SALSA2_scaffolds_out_JBAT.hic <(echo "assembly 171357824")

Note that the command, particularly the "assembly 171357824" part differs per assembly. The safest is to copy the JUICER_PRE CMD command found near the end of SALSA2_scaffolds_out_JBAT.log and replace ${juicer_tools} by /scripts/common/juicer_tools.1.9.9_jcuda.0.8.jar

After running all of these commands, you should have the following files:

* SALSA2_scaffolds_out_JBAT.liftover.agp
* SALSA2_scaffolds_out_JBAT.assembly
* SALSA2_scaffolds_out_JBAT.hic

Download SALSA2_scaffolds_out_JBAT.assembly and SALSA2_scaffolds_out_JBAT.hic to your local computer. These will be used as input to JBAT. The SALSA2_scaffolds_out_JBAT.liftover.agp will be used to generate a new FASTA file after manually correcting the scaffolds.

### Correcting scaffolds with JBAT

Start JBAT on your local computer, load SALSA2_scaffolds_out_JBAT.hic and SALSA2_scaffolds_out_JBAT.assembly into JBAT according to the following tutorial: https://youtu.be/Nj7RhQZHM18?feature=shared. Manually correct the scaffolds if necessary and generate a file called SALSA2_scaffolds_out_JBAT.review.assembly after you are done correcting, also according to the tutorial above. Copy the SALSA2_scaffolds_out_JBAT.review.assembly to the directory in which you generated the scaffolds (scaffolds_original_coordinates here)

Now, to generate a new FASTA file, run:

* yahs/juicer post -o SALSA2_scaffolds_out_JBAT SALSA2_scaffolds_out_JBAT.review.assembly SALSA2_scaffolds_out_JBAT.liftover.agp contigs.fa

This will generate the following output files:

* SALSA2_scaffolds_out_JBAT.FINAL.fa
* SALSA2_scaffolds_out_JBAT.FINAL.agp

SALSA2_scaffolds_out_JBAT.FINAL.fa contains the assembly after correction, which can be further checked for quality and polished. SALSA2_scaffolds_out_JBAT.FINAL.agp can be used to the generate a Hi-C map for the corrected assembly (see below).

### Generating Hi-C map for corrected scaffolds (optional)

This step is optional, but nice in order to show that the corrected assembly is supported by the Hi-C contact map. Starting in the scaffolds_original_coordinates directory:

* (yahs/juicer pre alignments.bam SALSA2_scaffolds_out_JBAT.FINAL.agp contigs.fa.fai | sort -k2,2d -k6,6d -T ./ --parallel=8 -S32G | awk 'NF' > alignments_sorted.txt.part) && (mv alignments_sorted.txt.part SALSA2_scaffolds_out_JBAT_alignments_sorted.txt)

* samtools faidx SALSA2_scaffolds_out_JBAT.FINAL.fa

* cut -f1,2 SALSA2_scaffolds_out_JBAT.FINAL.fa.fai > SALSA2_scaffolds_out_JBAT.chrom.sizes

* (java -jar -Xmx32G juicer_dir/scripts/common/juicer_tools.1.9.9_jcuda.0.8.jar pre SALSA2_scaffolds_out_JBAT_alignments_sorted.txt out.hic.part SALSA2_scaffolds_out_JBAT.chrom.sizes) && (mv out.hic.part SALSA2_scaffolds_out_JBAT_out.hic)

SALSA2_scaffolds_out_JBAT_out.hic can then be downloaded to your local computer, loaded in JBAT, and saved as an image to get an Hi-C map on the corrected assembly.
