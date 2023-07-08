# RNA-Seq-devs
 Building an RNA-seq workflow for Ion Torrent data 
## Basic introduction to Linux
There are lots of good introductions to Linux online. I recommend this one https://www.bioinf.wits.ac.za/courses/linux/handout.pdf it's purely for sentimental reasons as it written by my PhD supervisor.

## HPC2 cluster access and commands

How to log in to the HPC2 cluster https://www0.sun.ac.za/hpc/index.php?title=HOWTO_login

## RNA workflow development
We are essentially going to follow the RNA SOP provided by H3Africa https://h3abionet.github.io/H3ABionet-SOPs/RNA-Seq and we will split it into three phases as described in the workflow.


##PHASE1

[FASTQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/)

```
mkdir rawreads_fastqc
fastqc  --threads 6 -o rawreads_fastqc  TestData/*

````
Here we created the fastqc_raw  output directory using the mkdir command and then we run fastqc on the reads in the Testdata directory using 6 threads.


[multiqc](https://multiqc.info)

To aggregate the fastqc results into on report we use multiqc. 

```
multiqc  fastqc_rawreads  -o  multiqc_rawreads

```
As multiqc will create the output director we do not have to create one

