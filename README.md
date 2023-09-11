# RNA-Seq workflow development
 Building an RNA-seq workflow for Ion Torrent data 
## Basic introduction to Linux
There are lots of good introductions to Linux online. I recommend this one https://www.bioinf.wits.ac.za/courses/linux/handout.pdf it's purely for sentimental reasons as it was written by my PhD supervisor.

## HPC2 cluster access and commands

- How to log in to the HPC2 cluster https://www0.sun.ac.za/hpc/index.php?title=HOWTO_login

- I have found a great [presentation](https://www.sun.ac.za/english/faculty/science/sci-bioinformatics/Documents/Linux%20and%20HPC.pdf) from the Stellenbosch Center for Bioinformatics and Computational Biology on the HPC2 cluster, very relevant for our purpose.

### Useful commands: Check the HPC2 [HOWTO check up on jobs](https://www0.sun.ac.za/hpc/index.php?title=HOWTO_check_up_on_jobs) for details
<!-- # - quota -s # check disk quota -->
<!-- # - qstat # check the jobs queue -->
<!-- # - qstat -fx <JobID> # check job using JobID details -->
<!-- # - qdel  <JobID> # remove job from queue -->
<!-- # - pstat # overview of cluster usage -->


### Useful cluster related links
- Cluster job submission commands [[LINK]](https://uwaterloo.ca/math-faculty-computing-facility/resources/researchers/computing/sun-grid-engine-sge-batch-queuing-system/basic-sun-grid-engine-sge-job)

## RNA-Seq workflow development
We are essentially going to follow the RNA SOP provided by H3Africa https://h3abionet.github.io/H3ABionet-SOPs/RNA-Seq and we will split it into three phases as described in the workflow.


## PHASE 1

### Checking raw reads quality and trimming of reads

[FASTQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/)

```

mkdir rawreads_fastqc
fastqc  --threads 6 -o rawreads_fastqc  TestData/*

```

Here we created the rawreads_fastqc  output directory using the mkdir command and then we run fastqc on the reads in the Testdata directory using 6 threads.


[MULTIQC](https://multiqc.info)

To aggregate the fastqc results into on report we use multiqc. 

```

multiqc  rawreads_fastqc -o  rawreads_multiqc

```
As multiqc will create the output directory, we do not have to create one.

[TRIMMOMATIC](http://www.usadellab.org/cms/?page=trimmomatic)

The trimmomatic tool has two modes: PE (paired-end) and SE (single-end). We use the SE mode. The tools can only process one pair or one read at a time. This means you have it has to be run n times where n = number of reads.
Therefore, we loop through the files using a ```for``` loop. We create a directory for output files and use the ```basename``` command to extract the filename of the script. We also extract the filename without the extension from the associated filenames using ```basename``` command with the extension after the path. For readability we write command line arguments over multiple lines. This is achieved by using a backslash which is an escape character telling bash that the command continues on to the next line.

```

mkdir trimmed_reads
for SE_read in TestData/*.fastq
do
    #extract the read name without the extension
    read_basename=$(basename ${SE_read} .fastq)
    #extract the read name
    output_readname=$(basename ${SE_read})
    echo ${read_basename}
    trimmomatic SE \
	${SE_read} \
	trimmed_reads/${output_readname} \
	-threads 6 \
	-phred33 \
	-trimlog trimmed_reads/${read_basename}.log \
	LEADING:10 \
	TRAILING:10 \
	SLIDINGWINDOW:25:20 \
	MINLEN:50
done

```

After trimming we then run fastqc and multiqc to check the quality of our reads and to check if the trimming was effective.

### PHASE 1 in full

```
#!/bin/bash
#PBS -N RNA-seq-Phase1
#PBS -l select=1:ncpus=4:mem=8GB
#PBS -l walltime=12:00:00
#PBS -m be



rawread_dir=$1
output_dir=$2
threads=4

module load app/fastqc app/multiqc app/Trimmomatic/0.36

cd $PBS_O_WORKDIR

mkdir -p ${output_dir}/rawreads_fastqc

fastqc \
    --threads ${threads} \
    -o ${output_dir}/rawreads_fastqc \
    ${rawread_dir}/*.fastq


multiqc \
  ${output_dir}/rawreads_fastqc \
 -o  ${output_dir}/rawreads_multiqc


mkdir -p ${output_dir}/trimmed_reads

for SE_read in ${rawread_dir}/*.fastq
do
    #extract the read name without the extension
    read_basename=$(basename ${SE_read} .fastq)
    #extract the read name
    output_readname=$(basename ${SE_read})
    echo ${read_basename}
    trimmomatic SE \
    	${SE_read} \
    	${output_dir}/trimmed_reads/${output_readname} \
    	-threads ${threads} \
    	-phred33 \
    	-trimlog  ${output_dir}/trimmed_reads/${read_basename}.log \
    	LEADING:10 \
    	TRAILING:10 \
    	SLIDINGWINDOW:25:20 \
    	MINLEN:50
done


mkdir -p ${output_dir}/trimmed_fastqc

fastqc \
    --threads ${threads} \
    -o ${output_dir}/trimmed_fastqc \
    ${output_dir}/trimmed_reads/*.fastq


multiqc \
  ${output_dir}/trimmed_fastqc \
 -o  ${output_dir}/trimmed_multiqc

```

## PHASE 2

### Generating gene/transcript level counts

- The assembled transcriptome is available from NCBI and available on this [link](https://sra-download.ncbi.nlm.nih.gov/traces/wgs03/wgs_aux/GJ/ZM/GJZM01/GJZM01.1.fsa_nt.gz) under BioProject: [PRJNA835347](https://www.ncbi.nlm.nih.gov/bioproject/PRJNA835347)

- Important to note the extension on this file is .gz indicating that it is a compressed file created using gzip (GNU zip) compression algorithm. Not all tools support this file format so we have to uncompress this file with the following command ``` gunzip GJZM01.1.fsa_nt.gz``` 

We can now proceed to map our reads to the reference transcriptome with bowtie.

[BOWTIE 2](https://bowtie-bio.sourceforge.net/bowtie2/manual.shtml)

Before aligning our reads to the transcriptome reference we first have to build an index, this index is behind why Bowtie is ultrafast and memory efficient. We also create a directory for storing our references and indexes.

```
#!/bin/bash

#PBS -N Phase2-bowtie
#PBS -l select=1:ncpus=12:mem=24GB
#PBS -l walltime=12:00:00


threads=12
output_dir=/home/andhlovu/Phase2X
Ref=/home/andhlovu/DB_REF/Zcapensis_transcriptome/GJZM01.1.fsa_nt
index_basename=/home/andhlovu/DB_REF/Bowtie2/Z_capensis_transcriptome
trimmed_reads=/home/andhlovu/RNA-Seq-Phase1X/trimmed_reads



module load app/bowtie/2.3.4  app/samtools/1.17

cd $PBS_O_WORKDIR

mkdir -p $index_basename
bowtie2-build \
    --threads ${threads} \
    ${Ref} \
    ${index_basename}


mkdir -p $output_dir/BAM
mkdir -p $output_dir/SAM
for SE_read in ${trimmed_reads}/*.fastq
do
    base_fname=$(basename ${SE_read} .fastq)
    SAM_file=${base_fname}.sam
    BAM_file=${base_fname}.bam
    metrics=${base_fname}.metrics
    quick_stats=${base_fname}.quick_stats

    
    bowtie2 \
       --threads ${threads} \
       -x ${index_basename} \
       -U ${SE_read} \
       -S ${output_dir}/SAM/${SAM_file} \
       --met-file ${output_dir}/SAM/${metrics} 2> $output_dir/SAM/${quick_stats}          

    
   samtools \
       view \
       --threads ${threads} \
        ${output_dir}/SAM/${SAM_file} \
       -F 4 \
       -b \
       -o  ${output_dir}/BAM/${BAM_file}


    samtools \
	sort \
	${output_dir}/BAM/${BAM_file} \
	--threads  ${threads} \
	--reference ${Ref} \
	-o  ${output_dir}/BAM/${base_fname}_sorted.bam

    
    samtools \
	index \
	-@  ${threads} \
	${output_dir}/BAM/${base_fname}_sorted.bam 
         

    samtools \
	flagstat \
	${BAM} \
	--threads  ${threads}  > ${output_dir}/BAM/${base_fname}.flagstat    

done

```

The bowtie output file is in the Sequence Alignment/Map (SAM) Format. The format is controlled with a specified format. Our gene expression quantification is based on the data in this file, therefore it is important to have an understanding of the format to know what type of data can be extracted from the file. To do this, it is important to look at the [SAMv1](https://samtools.github.io/hts-specs/SAMv1.pdf) format specification document. The format specification may be a bit too technical luckily there are lots of tutorials online that explain the format in simpler terms.


[SALMON](https://salmon.readthedocs.io/en/latest/salmon.html#quantifying-in-alignment-based-mode)

Salmon is quasi-mapper and transcript quantifier. Instead of using the Salmon quasi-mapping and since we have already mapped our reads to the reference using bowtie, we will use the alignment mode ```salmon quant```. We will pass our *.bam files to Salmon, **not** the sorted files see [here](https://salmon.readthedocs.io/en/latest/salmon.html#using-salmon) for why. So sorted bam will have to be moved  into another location.  Salmon will produce a ```quant.sf``` and several auxiliary files in the output folder.The quant files and auxiliary files are required for downstream analysis so it is best to keep them together.

```
for bam in ${output_dir}/BAM/*.bam
do
base_name=$(basename ${bam} .bam)

salmon \
    quant \
    -l A \
    --gcBias \
    -a ${bam} \
    -t  ${Ref} \
    --threads ${threads} \
    -o ${base_name}

#can also rename our quant files
mv ${base_name}/quant.sf  ${base_name}/${base_name}.sf

done
```


## PHASE 3

### Perform statistical analysis to find differentially expressed genes


[TXIMPORT](https://bioconductor.org/packages/release/bioc/vignettes/tximport/inst/doc/tximport.html)

To use the salmon output in DESeq2 we first need to import our quant files into R using the ```tximport``` package. This also a new package [tximeta](https://bioconductor.org/packages/devel/bioc/vignettes/tximeta/inst/doc/tximeta.html) which extends the functionality of tximport; however, tximport is still supported. The major difference being tximport just requires the quant.sf file while tximeta requires the full salmon output with auxiliary file.

To import our quant file, we will make used of our metadata file which contains our readnames and additional experimental data. The metadata file should compiled using the sequencing report as the read ID may not necessarily contain the sample ID. So the metadata must be triple-checked to ensure controls and treatments are not mixed up.




>>>>>>> Stashed changes
