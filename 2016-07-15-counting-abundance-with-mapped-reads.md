#Estimating abundances in metagenomes

Authored by Jin Choi for EDAMAME2016 

## Summary

### Overarching Goal
* This tutorial will contribute towards an understanding of quantitative analyses of **metagenome data**
* It focuses on estimating abundances of reads to an assembled reference.

### Learning Objectives
* Understanding how to estimate abundances of reads in a representative gene reference
* Understanding read mapping
* Understanding mapping file formats
* Understanding how to use a mapping program (Bowtie2, samtools, bcftools)
* Apply reference mapping to assess read abundances and quantify gene presence

### Reference mapping is useful... 
* If you want to detect SNPs
* If you want to estimate abundance of genes in metagenomic or metatranscriptomic data

## Tutorial

### Install mapping software for this tutorial, Bowtie2 and SamTools.  BT2_HOME is the default name where Bowtie2 is installed.
Bowtie2 is a read mapping software.

```
cd 
wget http://sourceforge.net/projects/bowtie-bio/files/bowtie2/2.2.9/bowtie2-2.2.9-linux-x86_64.zip
unzip bowtie2-2.2.9-linux-x86_64.zip
mv bowtie2-2.2.9 BT2_HOME
```

Set up the path to installed software  (you need to set up path again if you are logged in later):

```
PATH=$PATH:~/BT2_HOME
export PATH
```

Install SamTools.  SamTools is a software that we use to work with files that are outputted by Bowtie2.  It is a software that is often used with mapping tools.  Mapping files are generally very big and get unwieldy because of their size, SamTools helps us deal with these large files in a memory efficient approaches but sometimes it adds a lot of steps at a cost of speeding up analysis.

```
sudo apt-get -y install samtools
```

Download data
```
cd ~/metagenome
wget https://s3.amazonaws.com/edamame/infant_gut.sub.tar.gz
tar -zxvf infant_gut.sub.tar.gz
```


## Do the mapping
Now let’s map all of the reads to the reference. Start by indexing the reference genome.  Indexing a reference stores in a memory efficient way on your computer:

```
cd ~/metagenome
bowtie2-build megahit_out/final.contigs.fa reference
```
Now, do the mapping of the raw reads to the reference genome (the -1 and -2 indicate the paired-end reads):
```
for x in SRR*_1.sub.fastq.gz;
  do bowtie2 -x reference -1 $x -2 ${x%_1*}_2.sub.fastq.gz -S ${x%_1*}.sam 2> ${x%_1*}.out;
done
```

This file contains all of the information about where each read hits our reference assembly contigs.

Next, index the reference genome with samtools.  Another indexing step for memory efficiency for a different tool.  In the mapping world, get used to indexing since the files are huge:

```
samtools faidx megahit_out/final.contigs.fa
```

Convert the SAM into a BAM file ([What is the SAM/BAM?](https://samtools.github.io/hts-specs/SAMv1.pdf)):

To reduce the size of a SAM file, you can convert it to a BAM file (SAM to BAM!) - put simply, this compresses your giant SAM file.

```
for x in *.sam;
  do samtools import megahit_out/final.contigs.fa.fai $x $x.bam;
done
```

Sort the BAM file - again this is a memory saving and sometimes required step, we sort the data so its easy to query with our questions:
```
for x in *.bam;
  do samtools sort $x $x.sorted;
done
```

And index the sorted BAM file:
```
for x in *.sorted.bam;
  do samtools index $x;
done
```

## Counting alignments
This command:
```
samtools view -c -f 4 SRR492065.sam.bam.sorted.bam
```
`-c` Instead of printing the alignments, only count them and print the total number. All filter options, such as -f, -F, and -q, are taken into account. `-f INT` Only output alignments with all bits set in INT present in the FLAG field. INT can be specified in hex by beginning with 0x (i.e. /^0x[0-9A-F]+/) or in octal by beginning with 0 (i.e. /^0[0-7]+/) [0]. 

will count how many reads DID NOT align to the reference (77608).

This command:

```
samtools view -c -F 4 SRR492065.sam.bam.sorted.bam
```
`-F INT` Do not output alignments with any bits set in INT present in the FLAG field. INT can be specified in hex by beginning with 0x (i.e. /^0x[0-9A-F]+/) or in octal by beginning with 0 (i.e. /^0[0-7]+/) [0].

will count how many reads DID align to the reference (122392).

And this command:

```
gunzip -c SRR492065_1.sub.fastq.gz | wc
```

will tell you how many lines there are in the FASTQ file (100,000). Reminder: there are four lines for each sequence.

This number give you idea how many sequences are mapped (%)

You can find this information in `.out` file also. 

## Make a summary of the counts 

```
for x in *.sorted.bam
do samtools idxstats $x > $x.idxstats.txt
done
```
We have some scripts that we use to process this file.
```
git clone https://github.com/metajinomics/mapping_tools.git
python mapping_tools/get_count_table.py *.idxstats.txt > counts.txt
less counts.txt
```

And there you are - you've created an abundance table.  Like an OTU count table, you can now use this file for statistical analyses when you do the mapping for multiple samples.

