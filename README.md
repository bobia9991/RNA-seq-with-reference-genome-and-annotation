# Tutorial for differential expression analysis using RNA-seq, a reference genome and pre-existing annotation. 

This repository is a tutorial for differential expression analysis using RNA-Seq data. The scripts are set up to run on UConn's Xanadu cluster, including Xanadu-specific SLURM headers and software modules. To run it on Xanadu, simply clone this repository and start submitting the scripts by following along with this readme. If you are interested in running it elsewhere, you'll need to install the relevant software and alter or remove the SLURM headers, but otherwise, the tutorial is self-contained and pulls the necessary data from ENSEMBL and NCBI. 

Commands should never be executed on the submit nodes of any HPC machine.  If working on the Xanadu cluster, you should use `sbatch scriptname` after modifying the script for each stage.  Basic editing of all scripts can be performed on the server with tools such as `nano`, `vim`, or `emacs`.  If you are new to Linux, please use [this](https://bioinformatics.uconn.edu/unix-basics) handy guide for the operating system commands.  In this guide, you will be working with common bioinformatic file formats, such as [FASTA](https://en.wikipedia.org/wiki/FASTA_format), [FASTQ](https://en.wikipedia.org/wiki/FASTQ_format), [SAM/BAM](https://en.wikipedia.org/wiki/SAM_(file_format)), and [GFF3/GTF](https://en.wikipedia.org/wiki/General_feature_format). You can learn even more about each file format [here](https://bioinformatics.uconn.edu/resources-and-events/tutorials/file-formats-tutorial/). If you do not have a Xanadu account and are an affiliate of UConn/UCHC, please apply for one **[here](https://bioinformatics.uconn.edu/contact-us/)**.


Contents
1. [ Overview ](#1-overview)
2. [ Accessing the Data ](#2-Accessing-the-expression-data-using-SRA-Toolkit-and-the-genome-via-ENSEMBL)  
3. [ Quality control ](#3-quality-control)
4. [ Aligning Reads to a Genome using `HISAT2` ](#4-aligning-reads-to-a-genome-using-hisat2)
5. [ Generating counts of fragments mapping to genes using `htseq-count` ](#5-Generating-counts-of-fragments-mapping-to-genes-using-htseq-count)
6. [ Pairwise differential expression with counts in `R` with `DESeq`2 ](#6-pairwise-differential-expression-with-counts-in-r-using-deseq2)
7. [ Getting gene annotations with `biomaRt` ](#7-getting-gene-annotations-with-biomart)
8. [ Gene ontology enrichment with `goseq` and `gProfiler` ](#8-gene-ontology-enrichment-with-goseq-and-gProfiler)  


## 1. Overview  

This tutorial uses a subset of [gene expression data](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE156460) from [Reid et al. 2016](https://doi.org/10.1126/science.aah4993), a study of evolutionary adaptation to industrial pollutants in the coastal fish *Fundulus heteroclitus*. The study consisted of an experimental exposure of embryonic fish from eight populations to the toxicant PCB-126. Here we will use exposed and control samples from two populations, one adapted to pollution (from the Atlantic Wood Industries superfund site in the Elizabeth River, VA; hereafter ER) and one not adapted (from King's Creek, VA, hereafter KC). Whole RNA was extracted, converted to cDNA and paired-end-sequenced on an Illumina HiSeq 1000. 

#### Setting up the work directory

This tutorial assumes you will be working on UConn's Xanadu cluster. So before beginning, you should log in to your account there via ssh. If you need an introduction to Xanadu, please see [here](https://bioinformatics.uconn.edu/resources-and-events/tutorials-2/xanadu/). 

To work through this tutorial, copy it to your home directory using the `git clone` command and entering the tutorial directory:

```bash
git clone git@github.com:CBC-UCONN/RNA-seq-with-reference-genome-and-annotation.git rnaseq-tutorial 
cd rnaseq-tutorial
```  

Once you clone the repository you can see the following folder structure: 

```
rnaseq-tutorial/
├── 01_raw_data
├── 02_quality_control
├── 03_index
├── 04_align
├── 05_align_qc
├── 06_count
├── 07_stringtie
├── 08_kallisto
├── images
├── LICENSE
├── r_analysis
├── README.md
└── setup.sh
```

#### SLURM scripts
The scripts provided are configured to use `SLURM`, the workload management software on the Xanadu cluster. Such software is necessary to manage and equitably distribute resources on large computing clusters with many users. You will run each script by submitting it to slurm using the command `sbatch`. For example, the first script will be submitted via the command `sbatch 01_fasterq_dump.sh`. 

Each script contains a header section that specifies computational resources (how many processors and what type, and memory) needed to run the job. SLURM then puts the job in a queue and then runs it when the resources become available. The header section looks like this:

```
#!/bin/bash
#SBATCH --job-name=JOBNAME
#SBATCH -n 1
#SBATCH -N 1
#SBATCH -c 1
#SBATCH --mem=1G
#SBATCH --partition=general
#SBATCH --qos=general
#SBATCH --mail-type=ALL
#SBATCH --mail-user=first.last@uconn.edu
#SBATCH -o %x_%j.out
#SBATCH -e %x_%j.err
```

Before beginning, you need to understand a few aspects of the Xanadu server. When first logging into Xanadu from your local terminal, you will be connected to a **submit node**. The submit node is meant to serve as an **interface** for users, and under no circumstances should you use it to do serious computation. If you do, the system administrators may kill your job and send you a mean e-mail about it. This may cause you to lose work, and worse, feel badly about yourself. On the submit node you may manage and inspect files, write and edit scripts, and do other very light duty work. To analyze data, you need to request access to one or more **compute nodes** through SLURM. This tutorial will not teach you how to configure the SLURM header. Therefore, before moving on, it would be helpful to learn about the topics covered in the [Xanadu tutorial](https://bioinformatics.uconn.edu/resources-and-events/tutorials-2/xanadu/) and our [guide to resource requests](https://github.com/CBC-UCONN/CBC_Docs/wiki/Requesting-resource-allocations-in-SLURM).


## 2. Accessing the expression data using SRA-Toolkit and the genome via ENSEMBL

Before we can get started, we need to get the data we're going to analyze. The sequence dataset from the experiment detailed above has been deposited in the [Sequence Read Archive (SRA)](https://www.ncbi.nlm.nih.gov/sra) at NCBI, a comprehensive collection of sequenced genetic data submitted by researchers. Sets of sequences (usually all the sequences from a given sample within an experiment) in the SRA are given a unique accession number. A set may be downloaded using `sratoolkit`. There are a variety of commands in `sratoolkit`, which you can read more about [here](https://www.ncbi.nlm.nih.gov/books/NBK158900/).  

An overview of the project data can be viewed [here](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE156460). 

We are going to download 19 samples corresponding to two toxicant exposure treatments (exposed and control) from one tolerant and one sensitive killifish population (ER and KC). 

| Accession   | Dose | Population      |
| ----------- | -    | --------------- |
| SRR12475447 | 0    | Elizabeth River |
| SRR12475448 | 0    | Elizabeth River |
| SRR12475449 | 0    | Elizabeth River |
| SRR12475450 | 0    | Elizabeth River |
| SRR12475451 | 200  | Elizabeth River |
| SRR12475452 | 200  | Elizabeth River |
| SRR12475453 | 200  | Elizabeth River |
| SRR12475454 | 2000 | Elizabeth River |
| SRR12475455 | 2000 | Elizabeth River |
| SRR12475456 | 2000 | Elizabeth River |
| SRR12475468 | 0    | King's Creek    |
| SRR12475469 | 0    | King's Creek    |
| SRR12475470 | 0    | King's Creek    |
| SRR12475471 | 0    | King's Creek    |
| SRR12475472 | 0    | King's Creek    |
| SRR12475473 | 200  | King's Creek    |
| SRR12475474 | 200  | King's Creek    |
| SRR12475475 | 200  | King's Creek    |
| SRR12475476 | 200  | King's Creek    |

Enter the directory `01_raw_data` by typing `cd 01_raw_data`. There are four files here: two scripts, [01_fasterq_dump.sh](/01_raw_data/01_fasterq_dump.sh) and [02_get_genome.sh](/01_raw_data/02_get_genome.sh), a list of accession numbers, [accessionlist.txt](/01_raw_data/accessionlist.txt), and a CSV table of metadata [metadata.txt](/01_raw_data/metadata.txt). 

### Downloading sequence data from the SRA

The first script downloads the data from the the SRA using `sratoolkit`. It contains two sections: a SLURM header and the block of code to be executed. 

The SLURM header, requesting 12 cpus and 15G of memory: 

```bash
#!/bin/bash
#SBATCH --job-name=fastqer_dump_xanadu
#SBATCH -n 1
#SBATCH -N 1
#SBATCH -c 12
#SBATCH --mem=15G
#SBATCH --partition=general
#SBATCH --qos=general
#SBATCH --mail-type=ALL
#SBATCH --mail-user=first.last@uconn.edu
#SBATCH -o %x_%j.out
#SBATCH -e %x_%j.err
```

The code (leaving out some comment lines):

```
module load parallel/20180122
module load sratoolkit/2.11.3

cat accessionlist.txt | parallel -j 2 fasterq-dump

ls *fastq | parallel -j 12 gzip
```

The first two lines load the software we use in the script. We're using 2 pieces of software here. `sratoolkit` and `parallel`. The `module load` commands make specific versions of each available on Xanadu. If you're working elsewhere, these will have to be installed and on your PATH. 

The second line downloads the data. There are a couple of things happening here. First, we're using a pipe (`|`). In bash scripting, the pipe is used to take the output of the command to the left of the pipe and send it to the command to the right of the pipe to be used as input. So we have two commands: `cat accessionlist.txt` and `parallel -j 2 fasterq-dump`. The first writes out our list of accessions. The second takes each entry on the list and runs `fasterq-dump` on it in parallel, downloading the sequence data. `-j 2` means download 2 accessions at a time. 

We will use this idiom repeatedly in the tutorial to run repetitive code in parallel. A useful tip with parallel, especially as commands get more complex: you can see the commands that would be executed without actually running them by adding the flag `--dryrun`. Try it:

```
cat accessionlist.txt | parallel --dryrun -j 2 fasterq-dump
```

In the third line, we use parallel again to gzip compress the fastq files. It's always a good idea to keep sequence data gzip compressed. It saves space, and pretty much every piece of software will read recognize and read gzipped files without issue. 

We're almost ready to run the script. Before running it, add your own e-mail address to the `--mail-user` option to receive notifications when the job starts, finishes, or ends with an error (or delete the line entirely if you don't want e-mails). 

When you're ready, you can execute the script by entering `sbatch 01_fasterq_dump.sh` in the terminal. This submits the job to the SLURM scheduler. It should take an hour or two to download all the data. 

Once the job is completed, you should see 57 sequence files in this directory with the suffix `fastq.gz`. Check this by typing `ls *fastq.gz | wc -l` on the command line. There should also be 2 log files produced by slurm with the suffixes `.err` and `.out`. These files will record messages and/or errors produced during execution of the script and should always be examined. 

There are three sequence files for each of our 19 samples. `_1.fastq.gz`, `_2.fastq.gz` and `fastq.gz`. These data were quality trimmed before being uploaded to the SRA, so even though we have paired-end sequence data, there are some single sequences left as a result. We're going to ignore the single end sequences going forward and only use the paired files ("_1.fastq.gz" and "_2.fastq.gz"). 

Lets have a look at at the contents of one of the fastq files:  

```
zcat SRR12475447_1.fastq.gz | head -n 12

@SRR1964642.1 FCC355RACXX:2:1101:1476:2162 length=90
CAACATCTCAGTAGAAGGCGGCGCCTTCACCTTCGACGTGGGGAATCGCTTCAACCTCACGGGGGCTTTCCTCTACACGTCCTGTCCGGA
+SRR1964642.1 FCC355RACXX:2:1101:1476:2162 length=90
?@@D?DDBFHHFFGIFBBAFG:DGHDFHGHIIIIC=D<:?BBCCCCCBB@BBCCCB?CCBB<@BCCCAACCCCC>>@?@88?BCACCBB>
@SRR1964642.2 FCC355RACXX:2:1101:1641:2127 length=90
NGCCTGTAAAATCAAGGCATCCCCTCTCTTCATGCACCTCCTGAAATAAAAGGGCCTGAATAATGTCGTACAGAAGACTGCGGCACAGAC
+SRR1964642.2 FCC355RACXX:2:1101:1641:2127 length=90
#1=DDFFFHHHHGJJJJJIIIJIJGIIJJJIJIJJGIJIJJJJIJJJJJJIJJJIJJJJJJJGIIHIGGHHHHHFFFFFDEDBDBDDDDD
@SRR1964642.3 FCC355RACXX:2:1101:1505:2188 length=90
GGACAACGCCTGGACTCTGGTTGGTATTGTCTCCTGGGGAAGCAGCCGTTGCTCCACCTCCACTCCTGGTGTCTATGCCCGTGTCACCGA
+SRR1964642.3 FCC355RACXX:2:1101:1505:2188 length=90
CCCFFFFFHHFFHJJJIIIJHHJJHHJJIJIIIJEHJIJDIJJIIJJIGIIIIJGHHHHFFFFFEEEEECDDDDEDEDDDDDDDADDDDD
```  

This writes out three sequence records. Each sequence record has four lines. The first is the sequence name, beginning with `@`. The second is the nucleotide sequence. The third is a comment line, beginning with `+`, and which here only contains the sequence name again (it is often empty). The fourth are [phred-scaled base quality scores](https://en.wikipedia.org/wiki/Phred_quality_score), encoded by [ASCII characters](https://drive5.com/usearch/manual/quality_score.html). Follow the links to learn more, but in short, the quality scores give the probability a called base is incorrect. 

### Downloading the genome, transcripts and annotation from ENSEMBL

The second script in this directory downloads the genomic resources we need for this analysis: ([02_get_genome.sh](/01_raw_data/02_get_genome.sh)). We're going to use data from the [ENSEMBL](https://www.ensembl.org/index.html) database. ENSEMBL is a resource for comparative genomics. It contains genomes and both structural and functional annotations for a large number of vertebrate genomes. Links to *Fundulus heteroclitus* data can be found [here](https://useast.ensembl.org/Fundulus_heteroclitus/Info/Index). 

This script is pretty simple, but it introduces another idiom that will be used repeated in following scripts. At the beginning of the script we set a "shell variable" to be used later in the script. Many scripts must repeatedly refer to files or directories with long names that make the code harder to read. We often replace these with shorter variables. This has two benefits: first, the script becomes easier to read and second if you need to edit a file name or path, you only need to change it in one place. 

In this case, we specify a path to a directory we're going to store all of our genomic resources in and then create the directory:

```bash
GENOMEDIR=../genome
mkdir -p $GENOMEDIR
```

`../genome`. Is a relative path. The `..` means go "up" one level in the directory tree. Try `ls ..` to see what this means. 

Next we download each of the necessary files using `wget` and decompress them using `gunzip`. Yes, we did say it's better to keep sequence files compressed, but many pieces of software expect genome and transcript sequences to be in plain text. 

Finally, we move all the files to the directory we just created:

```bash
mv Fundulus* $GENOMEDIR
```

Finally, `mv` moves files. The glob (`*`) is a wild card that matches any text, so any file that begins with "Fundulus" will be moved. `$GENOMEDIR` is how we access the shell variable we set at the top of the script. Note that `$` is necessary when accessing the variable, but not when setting it. 


## 3. Quality Control 

Here we use `fastp` to trim and produce HTML-formatted reports on characteristics of the sequences for each sample. `fastp` will trim low quality sequence and any residual adapter sequences (mostly a result of "read-through"). We'll also run `fastqc` which reports on a slightly different set of characteristics, and `multiqc` to aggregate the reports across samples. These reports will allow you to make a quick assessment of the overall quality of your sequences. 

Enter the directory `02_quality_control`. There are three scripts to run. The first script runs fastp ([01_fastp_trim.sh](/02_quality_control/01_fastp_trim.sh)). 

After the SLURM header we specify a few variables and create some directories to hold the results:

```bash
INDIR=../01_raw_data
REPORTDIR=fastp_reports
mkdir -p $REPORTDIR
TRIMDIR=trimmed_sequences
mkdir -p $TRIMDIR
```

Then we use our list of accessions and `parallel` to run `fastp` on all our samples:

```bash
cat $INDIR/accessionlist.txt | parallel -j 4 \
fastp \
	--in1 $INDIR/{}_1.fastq.gz \
	--in2 $INDIR/{}_2.fastq.gz \
	--out1 $TRIMDIR/{}_trim_1.fastq.gz \
	--out2 $TRIMDIR/{}_trim_2.fastq.gz \
	--json $REPORTDIR/{}_fastp.json \
	--html $REPORTDIR/{}_fastp.html
```

This parallel run is a little more complicated. To organize the code, here we use `\` to break up a single command over multiple lines. Parallel treats `{}` as a placeholder for the accessions it is reading from the pipe, and we construct the relevant filenames using it. Try adding `--dryrun` and running these lines to see the actual commands `parallel` is executing. 

`fastp` will trim the sequences and write new fastq.gz files to the `trimmed_sequences` directory and put the reports in `fastp_reports`. 

The second script ([02_fastqc.sh](/02_quality_control/02_fastqc.sh)) runs `fastqc` on the trimmed sequences, again using parallel:

```bash
INDIR=trimmed_sequences
REPORTDIR=fastqc_reports
mkdir -p $REPORTDIR

cat ../01_raw_data/accessionlist.txt | parallel -j 4 \
    fastqc --outdir $REPORTDIR $INDIR/{}_trim_{1..2}.fastq.gz
```

The final script ([03_multiqc.sh](/02_quality_control/03_multiqc.sh)) runs `multiqc` to aggregate HTML reports across samples. `-o` specifies the output directory `multiqc` will write reports to. 

```bash
multiqc -f -o fastp_multiqc fastp_reports

multiqc -f -o fastqc_multiqc fastqc_reports
```

[Transfer](https://bioinformatics.uconn.edu/resources-and-events/tutorials-2/data-transfer-2/) the multiqc reports to your local machine and view them in a web browser. You can use a xanadu node dedicated to file transfer: **transfer.cam.uchc.edu** and the unix utility `scp`. From a terminal on your local machine, you can copy the files as shown below, or use an FTP client with a graphical user interface such as FileZilla or Cyberduck: 

```bash
scp -r user-name@transfer.cam.uchc.edu:~/path/to/rnaseq-tutorial/02_quality_control/*multiqc .
```
The syntax is `scp x y`, meaning copy files `x` to location `y`. In this case "y" is `.`, which stands for your current working directory. 

You'll see that not much trimming was done. That's because these sequences were trimmed prior to being uploaded to NCBI. For more information on what's in the reports, check out the [the documentation for fastqc](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/). 




## 4. Aligning Reads to a Genome using `HISAT2`  

`HISAT2` is a fast and sensitive spliced aligner for mapping next generation sequencing reads against a reference genome. First we'll use it to build an index for the reference genome and then we'll use that to map the reads. 

### Building the Index:  

Why do we need an index? Genomes can be very large. A genome index allows the aligner to rapidly look up candidate alignment locations for each read rather than conduct an exhaustive search. 

Most aligners need to create their own indexes, so here we will use the `hisat2-build` module to make a HISAT index file for the genome. It will create a set of files with the suffix .ht2, these files together comprise the index. The commands to generate the index looks like this: 

```bash
OUTDIR=../genome/hisat2_index
mkdir -p $OUTDIR

GENOME=../genome/Fundulus_heteroclitus.Fundulus_heteroclitus-3.0.2.dna.toplevel.fa

hisat2-build -p 16 $GENOME $OUTDIR/Fhet
```  
Note here we tell `hisat2-build` to put the index files in our genome resources directory `../genome/hisat2_index` and to use the prefix `Fhet`. After you run the script that directory should contain these files:

```bash
genome/
├── Fundulus_heteroclitus.Fundulus_heteroclitus-3.0.2.105.gtf
├── Fundulus_heteroclitus.Fundulus_heteroclitus-3.0.2.cds.all.fa
├── Fundulus_heteroclitus.Fundulus_heteroclitus-3.0.2.cds.all.fa.fai
├── Fundulus_heteroclitus.Fundulus_heteroclitus-3.0.2.dna.toplevel.fa
├── Fundulus_heteroclitus.Fundulus_heteroclitus-3.0.2.dna.toplevel.fa.fai
└── hisat2_index
    ├── Fhet.1.ht2
    ├── Fhet.2.ht2
    ├── Fhet.3.ht2
    ├── Fhet.4.ht2
    ├── Fhet.5.ht2
    ├── Fhet.6.ht2
    ├── Fhet.7.ht2
    └── Fhet.8.ht2
```
The full script ([hisat2_index.sh](/03_index/hisat2_index.sh)) can be found in the folder `03_index`. Navigate there and submit it using `sbatch`. 


### Aligning the reads using `HISAT2`  

Once we have created the index, the next step is to align the reads to the reference genome with `HISAT2`. By default `HISAT2` outputs the alignments in SAM format. We won't go over the format in detail in this tutorial, but should you actually need to look at the alignment files, it would be helpful to read over the [format specification](https://samtools.github.io/hts-specs/SAMv1.pdf) or have a look the [wikipedia page](https://en.wikipedia.org/wiki/SAM_(file_format)). 

Raw SAM formatted files have two issues. First, they are uncompressed. Because of this they take up much more space than they need to and are slower to read and write. Second, `HISAT2` writes the alignments out in the same order it reads the sequences from the fastq file, but for downstream applications, they need to be sorted by **genome coordinate**. To solve these issues we're going to sort and compress the files to BAM format. 

We could do these three steps separately, but for computational and disk space efficiency, we'll use a _pipe_, as we did above, to send the results from `HISAT2` to `samtools` to sort the sequences, convert them to binary format and compress them. We can then use `samtools` to read or otherwise manipulate the resulting BAM fiels. We use pipes rather than writing intermediate files because it is much more efficient computationally, and requires less cleanup of unneeded intermediate files. 

Here's example code for aligning one sample:

```bash
hisat2 \
	-p 8 \
	-x $INDEX \
	-1 $INDIR/${SAMPLE}_trim_1.fastq.gz \
	-2 $INDIR/${SAMPLE}_trim_2.fastq.gz | \
samtools view -@ 8 -S -h -u - | \
samtools sort -@ 8 -T $SAMPLE - >$OUTDIR/$SAMPLE.bam
```  

The `|` is the pipe. It tells linux to use the output of the command to the left of the pipe as the input for the command to the right. You can chain together many commands using pipes. `samtools view` converts the SAM file produced by `hisat2` to uncompressed BAM. `-S` indicates the input is SAM format. `-h` indicates the SAM header should be written to the output. `-u` indicates that uncompressed BAM format should be written (no need to compress until the end). `-` indicates `samtools` should take input from the pipe. `samtools sort` sorts and compressed the file. `-T` gives a temporary file prefix. 

We're going to make this a little more complicated however, and use another method of paralellizing the job. Above we used the linux program `parallel`, here we're going to take advantage of a feature of SLURM called a "job array" to write one script, but have it submitted 19 times, once for each sample. You can read more about job arrays [here](https://github.com/CBC-UCONN/Job-Arrays-on-Xanadu), but we'll cover it briefly now. 

First, to make SLURM run a job array, you need to modify the header. 

```bash
#SBATCH -o %x_%A_%a.out
#SBATCH -e %x_%A_%a.err
#SBATCH --array=[0-18]%5
```

The first two lines specify the SLURM log files as in our previous header, except now each format includes a `%a`, which stands for the task number of the array. So now those log files will read something like `jobname_jobID_taskID.out`. The third line tells SLURM how many instances of the script (or "tasks") to run and how many it run at once. In this case we're running 19 tasks and we want to run them 5 at a time. Each task will be given an ID number from 0-18.  

In each instance of the job array, the variable SLURM_ARRAY_TASK_ID is set to the task ID, and we'll use that to pull out an accession number from our accession list. So the code will look like this:

```bash
INDIR=../02_quality_control/trimmed_sequences
OUTDIR=alignments
mkdir -p $OUTDIR

# hisat2 index
INDEX=../genome/hisat2_index/Fhet

# accession list location
ACCLIST=../01_raw_data/accessionlist.txt

# add 1 to the task ID (which we zero-indexed)
NUM=$(expr ${SLURM_ARRAY_TASK_ID} + 1)

# pull out a single sample accession
SAMPLE=$(sed -n ${NUM}p $ACCLIST)

# run hisat2
hisat2 \
	-p 8 \
	-x $INDEX \
	-1 $INDIR/${SAMPLE}_trim_1.fastq.gz \
	-2 $INDIR/${SAMPLE}_trim_2.fastq.gz | \
samtools view -@ 8 -S -h -u - | \
samtools sort -@ 8 -T $SAMPLE - >$OUTDIR/$SAMPLE.bam

# index bam files
samtools index $OUTDIR/$SAMPLE.bam
```

Because BAM files are large and we may want to access specific sections quickly, in the last line of the script we _index_ the bam files, just like we indexed the genome. We use `samtools` for this. Indexing creates a `.bam.bai` index file to accompany each BAM file. 

Once the mapping and indexing have been completed, the file structure is as follows, with two log files for each task and all of the .bam and .bai files written to the directory `alignments`. 

```bash
04_align/
├── align_4396649_0.err
├── align_4396649_0.out
├── align_4396649_10.err
├── align_4396649_10.out
├── align_4396649_11.err
├── align_4396649_11.out
├── align_4396649_12.err
├── align_4396649_12.out
├── align_4396649_13.err
├── align_4396649_13.out
├── align_4396649_14.err
├── align_4396649_14.out
├── align_4396649_15.err
├── align_4396649_15.out
├── align_4396649_16.err
├── align_4396649_16.out
├── align_4396649_17.err
├── align_4396649_17.out
├── align_4396649_18.err
├── align_4396649_18.out
├── align_4396649_1.err
├── align_4396649_1.out
├── align_4396649_2.err
├── align_4396649_2.out
├── align_4396649_3.err
├── align_4396649_3.out
├── align_4396649_4.err
├── align_4396649_4.out
├── align_4396649_5.err
├── align_4396649_5.out
├── align_4396649_6.err
├── align_4396649_6.out
├── align_4396649_7.err
├── align_4396649_7.out
├── align_4396649_8.err
├── align_4396649_8.out
├── align_4396649_9.err
├── align_4396649_9.out
├── alignments
└── align.sh
```  

To run the [full script](/04_align/align.sh), enter the `04_align` directory and submit the script by typing `sbatch align.sh` on the command line. When the job starts running, if you type `squeue`, you should see as many as 5 jobs running at once, with the rest pending in the queue. 

When `HISAT2` finishes aligning all the reads, it will write a summary which will be captured by SLURM in the files ending `.err`. 

An alignment summary for a single sample is shown below: 
```
25664909 reads; of these:
  25664909 (100.00%) were unpaired; of these:
    1114878 (4.34%) aligned 0 times
    23209585 (90.43%) aligned exactly 1 time
    1340446 (5.22%) aligned >1 times
95.66% overall alignment rate
```  

Let's have a look at a BAM file. If the BAM file is indexed, we can use `samtools` to access reads mapping to any part of the genome using the `view` submodule like this:

```bash
module load samtools/1.12
samtools view -H alignments/SRR12475447.bam | head
```

This line will print something like:
```bash
@HD     VN:1.0  SO:coordinate
@SQ     SN:KN805525.1   LN:6356336
@SQ     SN:KN805526.1   LN:6250690
@SQ     SN:KN811289.1   LN:6115441
@SQ     SN:KN811310.1   LN:5246820
@SQ     SN:KN811405.1   LN:5058772
@SQ     SN:KN811340.1   LN:5018176
@SQ     SN:KN811371.1   LN:4958163
@SQ     SN:KN811339.1   LN:4925094
@SQ     SN:KN811372.1   LN:4897111
```

Here we've requested that `samtools` return only the header section, which contains lots of metadata about the file, including all the contig names in the genome (each @SQ line contains a contig name). Each line begins with an "@" sign. The header can be quite large, especially if there are many contigs in your reference. 

To see the alignment records, try:

`samtools view alignments/SRR12475447.bam KN805525.1:20000-40000`

This will print to the screen all the reads that mapped to the genomic interval `KN805525.1:20000-40000`. 

You can use pipes and other linux tools to get basic information about these reads:

`samtools view alignments/SRR12475447.bam KN805525.1:20000-40000 | wc -l`

`wc -l` counts lines of text, so this command indicates that 87 reads map to this 20kb interval. 

### Alignment QC

Now that we have our alignments, we are going to generate some summary statistics on them to see if everything went well. We'll use two tools, `samtools` and `qualimap` and again aggregate the statistics produced by those tools with `multiqc` (we'll also put the main samtools stats in one big table with some bash code). 

The first script, [01_samtools_stats.sh](/05_align_qc/01_samtools_stats.sh), runs `samtools stats` on each bam file. The parallel command should look familiar now:

```bash
cat $ACCLIST | parallel -j 15 \
	"samtools stats $INDIR/{}.bam >$OUTDIR/{}.stats"
```

`samtools stats` output has tons of information in it, but we'll focus on the first table, e.g. for sample `SRR12475447`, which contains info on how many sequences there are, how many mapped, etc. 

```
grep "^SN" samtools_stats/SRR12475447.stats

SN      raw total sequences:    26685046
SN      filtered sequences:     0
SN      sequences:      26685046
SN      is sorted:      1
SN      1st fragments:  13342523
SN      last fragments: 13342523
SN      reads mapped:   22575888
SN      reads mapped and paired:        20233934        # paired-end technology bit set + both mates mapped
SN      reads unmapped: 4109158
SN      reads properly paired:  18740460        # proper-pair bit set
SN      reads paired:   26685046        # paired-end technology bit set
SN      reads duplicated:       0       # PCR or optical duplicate bit set
SN      reads MQ0:      109495  # mapped and MQ=0
SN      reads QC failed:        0
SN      non-primary alignments: 1015794
SN      total length:   2624444083      # ignores clipping
SN      total first fragment length:    1324788027      # ignores clipping
SN      total last fragment length:     1299656056      # ignores clipping
SN      bases mapped:   2228090187      # ignores clipping
SN      bases mapped (cigar):   2210488803      # more accurate
SN      bases trimmed:  0
SN      bases duplicated:       0
SN      mismatches:     15170309        # from NM fields
SN      error rate:     6.862875e-03    # mismatches / bases mapped (cigar)
SN      average length: 98
SN      average first fragment length:  99
SN      average last fragment length:   97
SN      maximum length: 101
SN      maximum first fragment length:  101
SN      maximum last fragment length:   101
SN      average quality:        36.0
SN      insert size average:    719.1
SN      insert size standard deviation: 1361.1
SN      inward oriented pairs:  9344564
SN      outward oriented pairs: 127493
SN      pairs with other orientation:   93441
SN      pairs on different chromosomes: 551365
SN      percentage of properly paired reads (%):        70.2
```

The samtools stats script also has some bash code for aggregating these tables for all samples into a single file. We won't cover it in detail here, but after you run the script you can have a look at it and try to figure out what it's doing. 

The second script, [02_qualimap.sh](/05_align_qc/02_qualimap.sh), runs `qualimap` on each bam file. In this case, we need to provide the GTF annotation file to so `qualimap` can count up how many reads map to annotated features. The command looks like this, again in parallel:

```bash
cat $ACCLIST | \
parallel -j 5 \
    qualimap \
        rnaseq \
        -bam $INDIR/{}.bam \
        -gtf $GTF \
        -outdir $OUTDIR/{} \
        --java-mem-size=2G  
```

Qualimap produces HTML-formatted output files you can download and view in your web browser. 

Finally, we'll aggregate all of these statistics using our `multiqc` script, [03_multiqc.sh](/05_align_qc/03_multiqc.sh). 

Move into the `05_align_qc` directory and submit the first two scripts. When they have finished, submit the multiqc script. When that is finished, download the reports and view them in a web browser. 

You should see that the mapping rates are in the mid-80s, and the rate of properly mapped read pairs is significantly lower. Of the mapped reads, around 20% map to regions of the genome not annotated as genes. This is likely because the fragmented nature of this genome assembly means reads pairs often map to different scaffolds and that the annotation is not very complete. 


## 5. Generating counts of fragments mapping to genes using `htseq-count`  

Now we will use the program `htseq-count` to count how many RNA fragments (i.e. read pairs) map to each annotated gene in the genome. Again, we'll use our accession list and the program `parallel`. We also need to provide our GTF formatted annotation so reads can be assigned to gene features. 

```bash
# accession list
ACCLIST=../01_raw_data/accessionlist.txt

# gtf formatted annotation file
GTF=../genome/Fundulus_heteroclitus.Fundulus_heteroclitus-3.0.2.105.gtf

# run htseq-count on each sample, up to 5 in parallel
cat $ACCLIST | \
parallel -j 5 \
    "htseq-count \
        -s no \
        -r pos \
        -f bam $INDIR/{}.bam \
        $GTF \
        > $OUTDIR/{}.counts"
```

- `-s no` indicates we're using an unstranded RNA-seq library. 
- `-r pos` tells `htseq-count` that our BAM file is coordinate sorted. 
- `-f bam` indicates that our input file is in BAM format. 


The above command should be repeated for all other BAM files as well. The full script is [htseq_count.sh](/06_count/htseq_count.sh) in the `06_count` directory. Go to that directory and submit the script now. This step will take a few hours. 

Once all the bam files have been counted, the following files will be found in the count directory.  
```bash
count/
├── counts
│   ├── SRR12475447.counts
│   ├── SRR12475448.counts
│   ├── SRR12475449.counts
│   ├── SRR12475450.counts
│   ├── SRR12475451.counts
│   ├── SRR12475452.counts
│   ├── SRR12475453.counts
│   ├── SRR12475454.counts
│   ├── SRR12475455.counts
│   ├── SRR12475456.counts
│   ├── SRR12475468.counts
│   ├── SRR12475469.counts
│   ├── SRR12475470.counts
│   ├── SRR12475471.counts
│   ├── SRR12475472.counts
│   ├── SRR12475473.counts
│   ├── SRR12475474.counts
│   ├── SRR12475475.counts
│   └── SRR12475476.counts
├── htseq_count_NNNNNNN.err
├── htseq_count_NNNNNNN.out
└── htseq_count.sh
```  

The count files are in `/06_count/counts`. Let's have a look at the contents of a counts file:

```bash
head SRR12475447.counts  
```

which will look like:  

```
ENSFHEG00000000002      0
ENSFHEG00000000003      974
ENSFHEG00000000004      0
ENSFHEG00000000005      2058
ENSFHEG00000000006      0
ENSFHEG00000000007      4185
ENSFHEG00000000008      0
ENSFHEG00000000009      0
ENSFHEG00000000010      0
ENSFHEG00000000011      3745
``` 

We see the layout is quite straightforward, with two columns separated by a tab. The first column gives the Ensembl gene ID, the second column is the number of mRNA fragments that mapped to the gene. These counts are the raw material for the differential expression analysis in the next section. 


## 6. Pairwise Differential Expression with Counts in `R` using `DESeq2`  

To identify differentially expressed (DE) genes, we will use the `R` package `DESeq2`, a part of the [Bioconductor](https://www.bioconductor.org/about/) project. After the counts have been generated, typical differential expression analyses can be done easily on a laptop, so we'll run this part of the analysis locally, instead of on the Xanadu cluster. 

There are a couple ways to download the appropriate files to your local computer. If you're on a mac or linux machine you can use "secure copy", `scp`. Close your Xanadu connection and run the following command in your terminal:  

```bash
scp user_name@transfer.cam.uchc.edu:/Path/to/count/*.counts /path/to/your/local/destination/directory  
```  

You may alternatively use an FTP client with a graphical user interface such as FileZilla or Cyberduck. `transfer.cam.uchc.edu` is the server used to transfer files. Regular login nodes cannot be used for file transfer. The transfer server should only be used to move files of small to moderate size (e.g < 1GB). For larger files, please use Globus (see tutorial on the CBC webpage). 

Typical DE analyses involve both a statistical analysis, which is used to rigorously identify genes of interest and their effect sizes, and data visualization, which is used both as a way of illustrating results and as a form of quality control. Even a quick examination of some of the plots we'll make below can reveal problematic samples or unexpected heterogeneity within treatment groups that could bias results, or to put things in a more positive light, inspire new analyses or interpretations. 

### Launching `R`

For this tutorial, we'll assume you're using `RStudio`, but however you launch R, it's always a good idea to ensure you begin a new project with a clean workspace. In `RStudio` we recommend you select "File > New Project" and start a new R project in whatever directory you choose. 

The following steps will use several different R packages. You'll need to make sure they're installed first. 

For differential expression and visualization:
- `DESeq2` needs to be installed through Bioconductor. See instructions [here](https://bioconductor.org/packages/release/bioc/html/DESeq2.html). 
- `apeglm` needs to be installed through Bioconductor. Instructions [here](https://bioconductor.org/packages/release/bioc/html/apeglm.html)
- `pheatmap` can be installed by typing `install.packages("pheatmap")`
- `tidyverse` a set of packages for data manipulation and visualization [tidyverse](https://tidyverse.tidyverse.org/) packages. `install.packages("tidyverse")`
- `ggrepel` an add-on package for ggplot that allows the plotting of labels so they don't overlap. `install.packages("ggrepel")`

For retrieving annotations and doing gene ontology enrichment:
- `biomaRt` needs to be installed through Bioconductor. Instructions [here](https://bioconductor.org/packages/release/bioc/html/biomaRt.html)
- `goseq` also needs to be installed through Bioconductor. Instructions [here](https://bioconductor.org/packages/release/bioc/html/goseq.html)


### The statistical analysis

All the following R code has been condensed into a single script "Full_DE_analysis.R" in the "r_analysis" directory of the tutorial repository. You can open that in `RStudio` rather than copying and pasting from here if you'd like. 

This first chunk of code will load the necessary R packages and assign values to some R objects we'll use to tell `DESeq2` how and where to read and write files. Note that in this example, the code is expecting to find the "count" directory in the parent of the project directory (that's what `directory <- "../06_count/count"` means). For this code to work, you'll have to edit the path to point to the directory containing *your* count data. 

```R
# Load the libraries we'll need in the following code:
library("DESeq2")
library("apeglm")
library("pheatmap")
library("tidyverse")
library("ggrepel")


# create an object with the directory containing your counts:
	# !!edit this to point to your own count file directory!!
directory <- "../06_count/counts"

# ensure the count files are where you think they are
list.files(directory)

sampleFiles <- list.files(directory, pattern = ".*counts")

```

Now we'll create a data frame that connects sample names, treatments, and count files. `DESeq2` has a function that can use that table to read and format the data so that it can be analyzed. 

```R
# load in metadata table
	# recode the population names to two letter abbreviations and the PCB-126 dosages to "control" and "exposed"
meta <- read.csv("../01_raw_data/metadata.txt") %>%
	mutate(population=str_replace(population, "Eliza.*", "ER")) %>%
	mutate(population=str_replace(population, "King.*", "KC")) %>%
	mutate(pcb_dosage = case_when(pcb_dosage == 0 ~ "control", pcb_dosage > 0 ~ "exposed"))

# ensure that sampleFiles and metadata table are in the same order

all( str_remove(sampleFiles, ".counts") == meta[,1] )

# now create a data frame with sample names, file names and treatment information. 
sampleTable <- data.frame(
	sampleName = meta$Run,
	fileName = sampleFiles,
	population = meta$population,
	dose = meta$pcb_dosage
	)

# look at the data frame to ensure it is what you expect:
sampleTable

# create the DESeq data object
ddsHTSeq <- DESeqDataSetFromHTSeqCount(
		sampleTable = sampleTable, 
		directory = directory, 
		design = ~ population * dose
		)
```

Here we need to point out that the last line of code in the above block is the formula that specifies the experimental design. Here we have two factors, population and dose, each with two levels (KC/ER and control/exposed). `population * dose` means we're telling `DESeq2` to estimate the effects for both population, dose, and and the population x dose interaction. The interaction term allows us to specifically ask about dose effects that are significantly different in each population, which was the main aim of this experiment. We could equivalently have specified this as `population + dose + population:dose`. 

Specifying design formulas (and extracting results) for complicated experiments can be tricky. [This](https://rstudio-pubs-static.s3.amazonaws.com/329027_593046fb6d7a427da6b2c538caf601e1.html) is a nice rundown of common examples, but if you have any doubts, you should 1) run your design formula and subsequent extraction of results by someone knowledgeable and/or 2) look at the counts of specific significant genes to ensure they match your expectations. 

Now we have a "DESeqDataSet" object (you can see this by typing `is(ddsHTSeq)`). By default, the creation of this object will order your treatments alphabetically, and the fold changes will be calculated relative to the first factor. If you would like to choose the first factor (e.g. the control), it's best to set this explicitly in your code. In this case, we want to use the control group from the KC population as the reference level, so we want to make sure "control" and "KC" are the first factor levels. Because they're sorted alphanumerically, "control" should already be, and we'll adjust the population factor below. 

```R
# To see the levels as they are now:
ddsHTSeq$population

# To replace the order with one of your choosing, create a vector with the order you want:
treatments <- c("KC","ER")

# Then reset the factor levels:
ddsHTSeq$population <- factor(ddsHTSeq$population, levels = treatments)

# verify the order
ddsHTSeq$population
```

As a next step, we can toss out some genes with overall low expression levels. We won't have statistical power to assess differential expression for those genes anyway. This isn't strictly necessary because `DESeq2` does _independent filtering_ of results, which will essentially ignore these very low expression genes, but if you have a lot of samples or a complex experimental design, it can speed up the analysis. 

```R
# what does expression look like across genes?

# sum counts for each gene across samples
sumcounts <- rowSums(counts(ddsHTSeq))
# take the log
logsumcounts <- log(sumcounts,base=10)
# plot a histogram of the log scaled counts
hist(logsumcounts,breaks=100)

# you can see the typically high dynamic range of RNA-Seq, with a mode in the distribution around 1000 fragments per gene, but some genes up over 1 million fragments. 

# get genes with summed counts greater than 20
keep <- sumcounts > 50

# keep only the genes for which the vector "keep" is TRUE
ddsHTSeq <- ddsHTSeq[keep,]
```

Ok, now we can do the statistical analysis. All the steps to fit the model are wrapped in a single function, `DESeq()`. 

```R
dds <- DESeq(ddsHTSeq)
```

When we run this function, a few messages will be printed to the screen:

```R
estimating size factors
estimating dispersions
gene-wise dispersion estimates
mean-dispersion relationship
final dispersion estimates
fitting model and testing
```

You can learn more in depth about the statistical model by starting with [this vignette](https://bioconductor.org/packages/devel/bioc/vignettes/DESeq2/inst/doc/DESeq2.html) and reading through some of the references to specific parts of the methods, but it will be helpful to explain a little here. 

- _estimating size factors_: This part of the analysis accounts for the fact that standard RNA-seq measures **relative abundances** of transcripts within each sample, not **absolute abundances** (i.e. transcripts/cell). Because of this, if libraries are sequenced to different depths, genes with identical expression levels will have different counts. It may seem that simply adjusting for sequencing depth could account for this issue, but changes in gene expression among samples complicate things, so a slightly more complex normalization is applied. 
- three dispersion estimate steps_: `DESeq2` models the variance in the counts for each sample using a _negative binomial distribution_. In this distribution the variance in counts among samples for one gene is determined by a _dispersion_ parameter. This parameter needs to be estimated for each gene, but for sample sizes typical to RNA-Seq (3-5 per treatment) there isn't a lot of power to estimate it accurately. The key innovation of packages like `DESeq2` is to assume genes with similar expression have similar dispersions, and so pool information across them to estimate the dispersion for each gene. These steps accomplish that using an "empirical Bayesian" procedure. Without getting into detail, what this ultimately means is that the dispersion parameter for a particular gene is a _compromise_ between the signal in the data for that gene, and the signal in other, similar genes. 
- _fitting model and testing_: Once the dispersions are estimated, this part of the analysis asks if there are treatment effects for each gene. In the default case, DESeq2 uses a _Wald test_ to ask if the log2 fold change between two treatments is significantly different than zero. 

Once we've done this analysis, we can start extracting nice clean tables of the results from the messy R object. The first thing we need to know is which results we'd like to extract. There are four sets of results we might want to see: 1) genes responding to exposure in KC, 2) genes responding to exposure in ER, 3) genes with population by exposure interactions and 4) baseline differences between populations. These results are each combinations of coefficients estimated in the model. 

Here we'll just focus on genes with significant interactions. For that, we only need to test whether the interaction term is significant. In the full R script we demonstrate how to extract the other results. It's important to specify the alpha value (for the adjusted p-values) you plan to use as your significance cutoff here, as it impacts the independent filtering procedure and thus the adjusted p-values. The default is `alpha=0.1`, which we're leaving unchanged. 

```R
# list coefficients
resultsNames(dds)

# get results table
res3 <- results(dds, name="populationER.doseexposed")

# get a quick summary of the table
summary(res3)

# check out the first few lines
head(res3)
```

In the results table there are six columns:

- _basemean_: the average of the normalized counts across samples. 
- _log2FoldChange_: the log2-scaled fold change. 
- _lfcSE_: standard error of log2 fold change. 
- _stat_: Wald statistic
- _pvalue_: raw p-value
- _padj_: A p-value adjusted for false discoveries. 

It's important to note here that the interaction term log2FoldChange is calculated with respect to the effect of treatment _in the reference level, KC_. So if a gene has a log2 fold change of 4 in KC exposed vs KC control, and the same gene has a log2 fold change of 0 in ER exposed vs ER control, then the interaction term log2 fold change will be -4. 

If we were interested in ranking the genes to look at, say, the top 20 most important, what would be the best factor? Log2 fold changes? Adjusted p-values? This is a tricky question. It turns out that both of these are not great because they are confounded with expression level. Genes with low expression can have unrealistically high fold change estimates (again because of low sample size), while genes with very high expression, but low fold changes, can have extremely low p-values. 

To deal with this, `DESeq2` provides a function for _shrinking_ the log2 fold changes. These shrunken log2 fold changes can be used to more robustly rank genes. Like the empirical Bayesian estimate of the dispersion parameter, this turns the log2 fold changes into a compromise between the signal in the data and a prior distribution that pulls them toward zero. If the signal in the data is weak, the log2 fold change is pulled toward zero, and the gene will be ranked lower. If the signal is strong, it resists the pull. This does not impact the p-values. 

We can get the shrunken log fold changes like this:

```R
# get shrunken log fold changes
res_shrink3 <- lfcShrink(dds,type="ashr",coef="populationER.doseexposed")

# plot the shrunken log2 fold changes against the raw changes:
data.frame(l2fc=res3$log2FoldChange, l2fc_shrink=res_shrink3$log2FoldChange, padj=res1$padj) %>%
	filter(l2fc > -5 & l2fc < 5 & l2fc_shrink > -5 & l2fc_shrink < 5) %>%
	ggplot(aes(x=l2fc, y=l2fc_shrink,color=padj > 0.1)) +
		geom_point(size=.25) + 
		geom_abline(intercept=0,slope=1, color="gray")

# get the top 20 genes by shrunken log2 fold change
arrange(data.frame(res_shrink3), log2FoldChange) %>% head(., n=20)
```

### RNA-Seq data visualization

There are several different visualizations we can use to illustrate our results and check to ensure they are robust. 

We'll start with the standard __Bland-Altman, or MA plot__, which is a high-level summary of the results. 

```R
plotMA(res_shrink3, ylim=c(-4,4))
```
MA-plots depict the log2 fold change in expression against the mean expression for each gene. In this case we're using the shrunken fold changes, but you can plot the raw ones as well. You don't typically learn much from an MA plot, but it does nicely illustrate that you can achieve significance (red dots) at a much smaller fold change for highly expressed genes. It can also be a first indication that a treatment causes primarily up or down-regulation. If the bulk of the points in the MA plot are shifted off of zero on the Y-axis, this is an indication that something has likely gone wrong with the normalization (calculation of scaling factors). 

Next we'll make a __volcano plot__, a plot of the negative log-scaled adjusted p-value against the log2 fold change. These aren't very useful, but they were presented in a lot of classic microarray-based gene expression studies. 

```R
# negative log-scaled adjusted p-values
log_padj <- -log(res_shrink3$padj,10)
log_padj[log_padj > 100] <- 100

# plot
plot(x=res_shrink3$log2FoldChange,
     y=log_padj,
     pch=20,
     cex=.5,
     col=(log_padj > 10)+1, # color padj < 0.1 red
     ylab="negative log-scaled adjusted p-value",
     xlab="shrunken log2 fold changes")

```


Next we'll plot the the results of a __principal components analysis (PCA)__ of the expression data. For each sample in an RNA-Seq analysis, there are potentially tens of thousands of genes with expression measurements. A PCA attempts to explain the variation in all those genes with a smaller number of synthetic variables called _principal components_. In an RNA-Seq analysis, we typically want to see if the PCs reveal any unexpected similarities or differences among samples in their overall expression profiles. A PCA can often clearly reveal outliers, sample labeling problems, or potentially problematic population structure. 


```R
# normalized, variance-stabilized transformed counts for visualization
vsd <- vst(dds, blind=FALSE)

# alternatively, using ggplot

dat <- plotPCA(vsd,returnData=TRUE,intgroup=c("population","dose"))

p <- ggplot(dat,aes(x=PC1,y=PC2,col=paste(population, dose)))
p <- p + geom_point() + 
	xlab(paste("PC1: ", round(attr(dat,"percentVar")[1],2)*100, "% variation explained", sep="")) + 
	ylab(paste("PC2: ", round(attr(dat,"percentVar")[2],2)*100, "% variation explained", sep="")) +
	geom_label_repel(aes(label=name))
p
```
![Expression PCA](/images/killi_pca.png)

Here we used a _variance stabilized transformation_ of the count data in the PCA, rather than the normalized counts. This is an attempt to deal with the very high range in expression levels (6 orders of magnitude) and the many zeroes in the count matrix, and is similar to, but a bit more robust than simpler approaches, such as simply adding 1 and log2 scaling the normalized counts. 

You can see that in our case, the first PC, which explains 43% of the total (scaled, variance stabilized) expression variance separates a single ER control sample from the rest. This is a strong indicator there may be something wrong with that sample. The embryo may have had issues or something could have gone wrong with the library preparation. It's worth digging into this further, and probably repeating the analysis with that sample excluded, though to keep the tutorial from getting too long we won't do that here. It's easy to edit the code at the beginning to exclude the sample. 

The second PC mostly separates ER from KC. The single KC control sample clustering with the other ER samples may indicate a sample labeling issue here as well, but it could just be that sample has a profile that is more ER-like. 

There's one more interesting foreshadow of results to come, the control and exposed samples from KC, but not ER, separate on PC2. It is sometimes, but not always, the case that treatment effects are clear in PCA plots. 

Next we'll make a heatmap of the top 30 most significant genes for the population x dose interaction. It's a bit tricky figuring out how to display expression values in a heatmap. Even with log-scaled counts the variation in overall expression can confound clustering, make the visualization messy, or both. Here we'll try plotting three different ways. First, we'll transform the counts using another method, the _regularized logarthim_. Then we'll grab the top 30 genes by shrunken log2 fold change. 

```R
# regularized log transformation of counts
rld <- rlog(dds, blind=FALSE)

# order gene names by absolute value of shrunken log2 fold change (excluding cook's cutoff outliers)
lfcorder <- data.frame(res_shrink3) %>%
  filter(!is.na(padj)) %>% 
  arrange(-abs(log2FoldChange)) %>% 
  rownames() 

# create a metadata data frame to add to the heatmaps
df <- data.frame(colData(dds)[,c("population","dose")])
  rownames(df) <- colnames(dds)
  colnames(df) <- c("population","dose")
```

Now we'll plot just the regularized counts. 

```R
# use regularized log-scaled counts
pheatmap(
  assay(rld)[lfcorder[1:30],], 
  cluster_rows=TRUE, 
  show_rownames=TRUE,
  cluster_cols=TRUE,
  annotation_col=df
  )
```
![regularized count heatmap](/images/heatmap1.jpg)

This is pretty messy. One issue with this plot is that genes with overall very high expression are clustering together rather than those with similar fold changes. Next we can scale by _baseMean_, effectively dividing all the counts by the mean across the whole experiment. 

```R
# re-scale regularized log-scaled counts by baseMean (estimated mean across all samples)
pheatmap(
  assay(rld)[lfcorder[1:30],] - log(res1[lfcorder[1:30],"baseMean"],2), 
  cluster_rows=TRUE, 
  show_rownames=TRUE,
  cluster_cols=TRUE,
  annotation_col=df
  )
```
![scaled by baseMean heatmap](/images/heatmap2.jpg)

This might normally be a good choice, but we have one gene, Cyp1A (ENSFHEG00000010510) that has such a massive fold change in KC that it dominates the plot. Also, _baseMean_ doesn't really have any natural meaning to us. Lastly we can try scaling by the average expression level in the control samples in our reference population, KC. 

![scaled by KC-control heatmap](/images/heatmap3.jpg)

I think this one displays the data the best. We still don't get totally clean clustering by expression profile, but to my eye this best shows a few features of the data: 1) for these genes with significant interactions, the pattern is that exposed KC individuals are showing a strong response to treatment, but ER is not. 2) Almost all of the response to treatment in KC is up-regulation, with a smaller subset of genes being down-regulated. Our outlier sample SRR12475449, is an outlier even for important genes, suggesting it should be removed. 

Finally, when you've got DE genes, it's always a good idea to at least spot check a few of them to see if the normalized counts seem consistent with expectations. This can reveal a few different problems. First, although `DESeq2` attempts to manage outliers, they can sometimes drive the signal of DE. If you're interested in hanging a bunch of interpretation on a few genes, you should check them. Second, unexpected heterogeneity within treatment groups can also drive signal. This is not the same as having outliers, and it may indicate problems with experimental procedures, or possibly something biologically interesting. Often this will also show up in a PCA or heatmap.

```R
plotCounts(dds, gene=lfcorder[1], intgroup=c("population","dose"))
```

Here we've plotted the gene with the largest shrunken log2 fold change. 


## 7. Getting gene annotations with `biomaRt`


BioMart is software that can query a variety of biological databases. `biomaRt` is an R package we can use to build those queries. Here we'll use `biomaRt` to retrieve gene annotations from the Ensembl database. 

```R
# load biomaRt
library(biomaRt) 


##############################
# Select a mart and dataset
##############################

# see a list of "marts" available at host "ensembl.org"
listMarts(host="ensembl.org")

# create an object for the Ensembl Genes v100 mart
mart <- useMart(biomart="ENSEMBL_MART_ENSEMBL", host="ensembl.org")

# occasionally ensembl will have connectivity issues. we can try an alternative function:
	# select a mirror: 'www', 'uswest', 'useast', 'asia'
	# mart <- useEnsembl(biomart = "ENSEMBL_MART_ENSEMBL", mirror = "useast")

# see a list of datasets within the mart
	# at the time of writing, there were 203
listDatasets(mart)

# figure out which dataset is the killifish
	# be careful using grep like this. verify the match is what you want
searchDatasets(mart,pattern="Mummichog")

# there's only one match, get the name
killidata <- searchDatasets(mart,pattern="Mummichog")[,1]

# create an object for the killifish dataset
killi_mart <- useMart(biomart = "ENSEMBL_MART_ENSEMBL", host = "ensembl.org", dataset = killidata)

# if above there were connectivity issues and you used the alternative function then:
	# select a mirror: 'www', 'uswest', 'useast', 'asia'
	# killi_mart <- useEnsembl(biomart = "ENSEMBL_MART_ENSEMBL", dataset = killidata, mirror = "useast")


#########################
# Query the mart/dataset
#########################

# filters, attributes and values

# see a list of all "filters" available for the mummichog dataset.
	# at the time of writing, over 300
listFilters(killi_mart)

# see a list of all "attributes" available
	# 129 available at the time of writing
listAttributes(mart = killi_mart, page="feature_page")

# we can also search the attributes and filters
searchAttributes(mart = killi_mart, pattern = "ensembl_gene_id")

searchFilters(mart = killi_mart, pattern="ensembl")

# get gene names and transcript lengths when they exist
ann <- getBM(filter="ensembl_gene_id",value=rownames(res1),attributes=c("ensembl_gene_id","description","transcript_length"),mart=killi_mart)

# pick only the longest transcript for each gene ID
ann <- group_by(ann, ensembl_gene_id) %>% 
  summarize(.,description=unique(description),transcript_length=max(transcript_length)) %>%
  as.data.frame()

# get GO term info
  # each row is a single gene ID to GO ID mapping, so the table has many more rows than genes in the analysis
go_ann <- getBM(filter="ensembl_gene_id",value=rownames(res1),attributes=c("ensembl_gene_id","description","go_id","name_1006","namespace_1003"),mart=killi_mart)

# put results and annotation in the same table
res_ann3 <- cbind(res_shrink3,ann)
```

## 8. Gene ontology enrichment with `goseq` and `gProfiler`

Once we have a set of differentially expressed genes, we want to understand their biological significance. While it can be useful to skim through the list of DE genes, looking for interesting hits, taking a large-scale overview is often also helpful. One way to do that is to look for enrichment of functional categories among the DE genes. There are many, many tools to accomplish that, using several different kinds of functional annotations. Here we'll look at enrichment of [gene ontology](http://geneontology.org/) terms. The Gene Ontology (GO) is a controlled vocabulary describing functional attributes of genes. Above, we retrieved GO terms for the genes we assayed in our experiment. Here we will ask whether they are over- (or under-) represented among our DE genes. 

We're going to use two tools: a Bioconductor package, `goseq` and a web-based program, `gProfiler`. 

`goseq` can pull annotations automatically for some organisms, but not _L. crocea_, so we'll need to put together our input data. We need:

- A vector of 1's and 0's indicating whether each gene is DE or not. 
- A vector of transcript lengths for each gene (the method tries to account for this source of bias). 
- A table mapping GO terms to gene IDs (this is the object `go_ann`, created above)

```R
library(goseq)

# 0/1 vector for DE/not DE, excluding genes with no p-values
de <- as.numeric(res3$padj[!is.na(res3$padj)] < 0.1)
names(de) <- rownames(res3)[!is.na(res3$padj)]

# length of each gene,  excluding genes with no p-values
len <- ann[[3]][!is.na(res3$padj)]
```

Now we can start the analysis:

```R
# first try to account for transcript length bias by calculating the
# probability of being DE based purely on gene length
pwf <- nullp(DEgenes=de,bias.data=len)

# use the Wallenius approximation to calculate enrichment p-values
GO.wall <- goseq(pwf=pwf,gene2cat=go_ann[,c(1,3)])

# do FDR correction on p-values using Benjamini-Hochberg, add to output object
GO.wall <- cbind(
  GO.wall,
  padj_overrepresented=p.adjust(GO.wall$over_represented_pvalue, method="BH"),
  padj_underrepresented=p.adjust(GO.wall$under_represented_pvalue, method="BH")
  )
```

And that's it. We now have our output, and we can explore it a bit. Here we'll just pick the one of the top enriched GO terms, pull out all the corresponding genes and plot their log2 fold-changes. 

```R
# explore the results

head(GO.wall)

# identify ensembl gene IDs annotated with to 2nd from top enriched GO term
  # this category contains Aryl hydrocarbon receptor. 
g <- go_ann$go_id==GO.wall[2,1]
gids <- go_ann[g,1]

# inspect DE results for those genes
res_ann3[gids,] %>% data.frame()


# plot log2 fold changes for those genes, sorted
ord <- order(res_ann3[gids,]$log2FoldChange)
plot(res_ann3[gids,]$log2FoldChange[ord],
     ylab="l2fc of genes in top enriched GO term",
     col=(res_ann3[gids,]$padj[ord] < 0.1) + 1,
     pch=20,cex=.5)
abline(h=0,lwd=2,lty=2,col="gray")

```

We can execute a similar analysis using the web server `gProfiler`. First, get a list of DE gene IDs:

```R
cat(rownames(res3)[res3$padj < 0.1])
```
Copy the text to your clipboard. Then visit [gProfiler](https://biit.cs.ut.ee/gprofiler/gost). Select _Fundulus heteroclitus_ as the organism. Paste the output into the query window and press "Run Query". Explore the results. We should see extremely similar enrichment terms. 

A major difference between these analyses is the background set of genes considered. We only give `gProfiler` our list of DE genes. It compares those against the total set of gene annotations in _F. heteroclitus_ (nearly 24,000 genes). `goseq`, by contrast, is using only the set of genes we included in the analysis (only about 20,000). 


## Writing out results

Finally, we can write somes results out to files like this:

```R
# set a prefix for output file names
outputPrefix <- "killifish_DESeq2"

# save data results and normalized reads to csv
resdata <- merge(
  as.data.frame(res_shrink), 
  as.data.frame(counts(dds,normalized =TRUE)), 
  by = 'row.names', sort = FALSE
)
names(resdata)[1] <- 'gene'

write.csv(resdata, file = paste0(outputPrefix, "-results-with-normalized.csv"))

# send normalized counts to tab delimited file for GSEA, etc.
write.table(
  as.data.frame(counts(dds),normalized=T), 
  file = paste0(outputPrefix, "_normalized_counts.txt"), 
  sep = '\t'
)

```



