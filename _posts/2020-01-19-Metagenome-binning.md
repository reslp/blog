---
layout: post
title: A script to run and combine results of Metagenome binners
---

# Binning metagenomes

I have recently worked with a larger number of [lichen](https://en.wikipedia.org/wiki/Lichen) metagenomes. Lichens are symbiotic systems containing fungi, algae and bacteria. These different organisms are so intertwined with each other that they usually can't be seperated prior to DNA extraction. As a results high-throughput sequencing data produced from lichens usually contains DNA from different organisms and organells. This impacts downstream analyses and it is therefore often desired to seperate DNA from different origanisms computationally. This is not a trivial task. Common approaches use coverage information, k-mer profiles, GC content or sequence similarity to seperate sets of sequences (hopefully) belonging to single organisms. Such sequence sets are often called [bins](https://en.wikipedia.org/wiki/Binning_(metagenomics)). 
Seperating contaminants from the organism of interest is a common problem and luckily many different programs exist to accomplish this. Some examples include [MetaBat](https://bitbucket.org/berkeleylab/metabat/src/master/), [Concoct](https://github.com/BinPro/CONCOCT) or [blobtools](https://blobtools.readme.io/docs), but many more exist.
The sequence bins created with any of these programs can then be fruther analysed to calculate standard metrics such as N50 and total length using tools like [QUAST](http://quast.sourceforge.net/quast) and gene completeness with software like [BUSCO](https://busco.ezlab.org/) 


# Repetition is boring

When applying these programs to my datasets I realized that I do the same steps over and over again. A typical task would be to run blobtools on a genome (with all necessary steps), extract bins with a custom script, run QUAST and BUSCO on a set of sequences to evaluate the quality of the bin. Repeat with next genome.


Additionally, individual binners can depend on other software to work. Blobtools for example requires a read mapper, a [taxdump](https://ncbiinsights.ncbi.nlm.nih.gov/2018/02/22/new-taxonomy-files-available-with-lineage-type-and-host-information/) file from NCBI, several different python modules etc. These dependencies can make it frustrating to set up and run a program on different computers or clusters. 

The desire to automate repetitive tasks compared with wanting a solution which is easily portable to other computers lead me to write a small bash script using [Docker](https://www.docker.com) containers with different metagenome binners.

# A tool to combine different binners

I wrote [binner](http://github.com/reslp/binner) to solve the above mentioned challenges. Binner is a BASH script which can run different metagenome binning programs. Currently it works with [Maxbin2](https://sourceforge.net/projects/maxbin2/), [MetaBat](https://bitbucket.org/berkeleylab/metabat/src/master/), [Concoct](https://github.com/BinPro/CONCOCT) and [blobtools](https://blobtools.readme.io/docs). Every program is containerized using Docker, making a *nix based system with a working Docker installation the only requirements to run binner. You also don't have to know how to run Docker, binner will take care of this for you.

In terms of input files you will need a (metagenome) assembly and a set of Illumina reads in FASTQ format.

When Docker is installed, running binner is as easy as cloning the repository, making the script executable and running it:

```
$ git clone git clone https://github.com/reslp/binner.git
$ cd binner
$ chmod +x binner
$ ./binner -h
Welcome to binner. A script to quickly run metagenomic binning software using Docker.

Usage: binner [-v] [-a <assembly_file>] [-f <read_file1>] [-r <read_file2>] [-m maxbin,metabat,blobtools,concoct] [-t nthreads] [[--diamonddb=/path/to/diamonddb --protid=/path/to/prot.accession2taxid]] [-q] [-b [--buscosets=set1,set2]]

Options:
	-a <assembly_file> Assembly file in FASTA format
	-f <read_file1> Forward read file in FASTQ format (can be gzipped)
	-r <read_file2> Reverse read file in FASTQ format (can be gzipped)
		IMPORTANT: Currently the assembly and read files need to be in the same directory which has to the directory in which binner is run.
	-m <maxbin,metabat,blobtools,concoct> specify binning software to run.
	   Seperate multiple options by a , (eg. -m maxbin,blobtools).
	-t number of threads for multi-threaded parts
	-q run QUAST on the binned sets of contigs
	-b run BUSCO on the binned sets of contigs. See additional details below.
	--multiqc Run multiqc after all binning steps to create a summary report on all the bins. This should be used together with -q, -b or both.

	-v Display program version

Options specific to blobtools:
	The blobtools container used here uses diamond instead of blast to increase speed.
	Options needed when blobtools should be run. The blobtools container used here uses diamond instead of blast to increase speed.
  	--diamonddb=	full (absolute) path to diamond database
  	--protid= 	full (absolute) path to prot.accession2taxid file provided by NCBI

Options needed when BUSCO analysis of contigs should be performed:
	The blobtools container used here uses diamond instead of blast to increase speed.
	Options needed when blobtools should be run. The blobtools container used here uses diamond instead of blast to increase speed.
		--buscosets=	BUSCO sets which should be tested. This will be run for each set of contigs. Individual sets should be comma
				separated. eg. --buscosets=fungi_odb9,bacteria_odb9,insects_odb9 .
				Running this assumes that folders with the busco sets exist in the current working directory. They should have
				the same name as passed to the --buscosets command. If they are not found binner will try to download them
				from the BUSCO website.
```

# Run each binner seperately

Individual binners can be run independently. Here is an example of binner running Maxbin2:

```
$ binner -a metagenome.fasta -f forward_readfile.fq -r reverse_readfile.fq -m metabat
```

For more examples look at the README file of the [binner repository](http://github.com/reslp/binner).

# Combine results with multiqc

One additional feature of binner is the ability to evaluate bins extracted from different programs with QUAST and BUSCO and to aggregate these results using [multiqc](https://multiqc.info).
This can be achieved like this:

```
$ binner -a metagenome.fasta -f forward_readfile.fq -r reverse_readfile.fq -m maxbin,blobtools,concoct -b -q --buscosets=fungi_odb9,dikarya_odb9 -b /path/to/diamonddb -p /path/to/prot.accession2taxid --multiqc
```

# More to come

Right now binner is working quite well and I have incorporated it in to my daily workflow. However I can still see many ways to improve it and I plan to do so in the future. Some things on my list for future versions include:

- incorporate additional binners
- add more flexibility to tune parameters of individual binners 
- find consensus bins accross different binners
- add ability to add use other alignment tools in blobtools apart from diamond

