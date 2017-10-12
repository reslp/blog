---
layout: post
title: My MAKER2 pipeline
---
In this post I describe how I usually use MAKER2 to predict genes in the fungal genomes I have sequenced. This post does not cover installing MAKER, which I will save for another time. 

# Overview

My gene calling approach is to a large part derived from the very nice paper by [Campbell et al. 2014](https://www.ncbi.nlm.nih.gov/pubmed/25501943) which contains a lot of information on howto run MAKER with different aims. When I first used MAKER to annotate some *de-novo* sequenced fungale genomes I realized that it can be tricky to provide the correct input files and configuration options for the different gene callers MAKER uses. I had to distill needed information from google groups and other online forums and publications. The protocol provided here is an attempt to provide all the needed information to run MAKER in a single place.

*Note: The \$ sign at the beginning of the code chunks represents the command prompt. It does not need to by typed in.*

# The workflow

[MAKER](http://www.yandell-lab.org/software/maker.html) runs multiple independent gene callers and creates consensus predictions based on collective evidence. For this to work properly MAKER needs to be run several times accumulating more and more gene predicting evidence created along the way. I usually use the three gene predictors [SNAP](http://korflab.ucdavis.edu/software.html), [AUGUSTUS](http://augustus.gobics.de) and [GeneMark](http://opal.biology.gatech.edu/genemark/). Here is my basic workflow at a glance:

1. Run MAKER with EST and/or protein evidence 
2. Train SNAP based on results from Step 1 
3. Run MAKER with the training results from Step 2
4. Train SNAP a second time with results from Step 3
5. Run MAKER with the training results from Step 4
6. Train AUGUSTUS
7. Run again like 5 but include AUGUSTUS training file and set keep_preds=1
8. Train GeneMark
9. Run MAKER with SNAP, AUGUSTUS and GeneMark predictors
10. Create Maker standard gene set (not covered in this post)

# Step 1

When trying to predict genes in a newly assembled genome a good starting point for first preliminary predictions can be RNASeq data, ESTs from the same organism and already predicted proteins from the same or a closely related organism. The initial gene models created in this step may later serve to train several [HMM](https://en.wikipedia.org/wiki/Hidden_Markov_model) based gene predictors.

The first thing is to create new control files for your MAKER run. The control files tell MAKER how to run and where to find additional software such as different gene callers.

In the desired folder for your MAKER run type:

	$ maker -CTL

This will create three files:

* `maker_bopts.ctl` containing settings for BLAST and Exonerate.
* `masker_exe.ctl` with all the paths to different executables used by MAKER on your system.
* `maker_opts.ctl` is the file controlling MAKERs running behavior. This file will be changed before every MAKER run.

Open the `maker_opts.ctl` file in your favorite text editor (eg. nano). The lines you will want to change now are:

```genome=/home/philipp/data/my_assembly.fasta #genome sequence (fasta file or fasta embeded in GFF3 file)```
the path to your organisms genome assembly,

```organism_type=eukaryotic #eukaryotic or prokaryotic. Default is eukaryotic```
the type of organism and

```est= #set of ESTs or assembled mRNA-seq in fasta format```
and
```protein= #protein sequence file in fasta format (i.e. from mutiple oransisms)```

given that you have both protein and RNA seq evidence ready. If you have only one of the two simply leave the other line as it is. After changing, the lines may look something like this:

```est=/home/philipp/data/rnaseq/Trinity.fasta #set of ESTs or assembled mRNA-seq in fasta format```

```protein=/home/philipp/data/proteins/predicted_proteins.fasta #protein sequence file in fasta format (i.e. from mutiple oransisms)```

Also set the following flags to enable gene prediction solely on RNA and protein evidence:

`
est2genome=1 #infer gene predictions directly from ESTs, 1 = yes, 0 = no
`
`
protein2genome=1 #infer predictions from protein homology, 1 = yes, 0 = no
`

In the directory containing the control files you can now execute MAKER by typing: 

	$ maker


# Step 2

Once the MAKER run from Step 1 is done it is time to train the first gene predictor [SNAP](http://korflab.ucdavis.edu/software.html) with the results acquired in Step 1.

For this we need to extract the results from the first MAKER run. MAKER results are stored in a directory with the name of your assembly file plus .maker.output. In this directory execute:

```
$ gff3_merge -d my_assembly_datastore_index.log 
```
Change the file name to whatever the name of your datastore_index.log file is. This command creates a GFF3 file containing all the preliminary gene predications created in Step 1.

We will now generate the necessary files for training SNAP:

Execute:

 ```
 $ maker2zff my_assembly.all.gff
 ```
 
 This script (which comes with MAKER) will create two files: `genome.ann` and `genome.dna` which contain information about the gene sequences (such as exons and introns) as well as the actual DNA sequences.
 
Let's validate them with fathom (comes with SNAP) to detect erroneous predictions:

```
$ fathom genome.ann genome.dna -validate > snap_validate_output.txt
```

After completing this command you will see if there are any erroneous predictions. If there are we will remove them.

In this case execute the following command to identify the erroneous model(s).

```
$ cat snap_validate_output.txt | grep "error" 
```

This will give you output similar to this:
`scaff40651: MODEL6064 16223 17246 5 + errors(1): cds:internal_stop warnings(2): split-start exon-1:short(1)`

So lets remove the gene model from the file containing all gene models by running:

```
$ grep -vwE "MODEL6064" genome.ann > genome.ann2
```

Rerunning fathom should now show no errors:

```
$ fathom genome.ann2 genome.dna -validate 
```

Now that the files are error free, lets create the remaining input files for training SNAP. Execute:

	$ fathom genome.ann genome.dna -categorize 1000
	$ fathom uni.ann uni.dna -export 1000 -plus 
	$ forge export.ann export.dna

Please refer to the SNAP documentation for details about these commands. Now we have everything ready to train SNAP with hmm-assembler (which comes with SNAP):

	$ hmm-assembler.pl my_genome . > my_genome.hmm


# Step 3
	
We will now link the newly created SNAP hmm file in the MAKER control file. Change the following line in the `maker_opts.ctl` file:

```snaphmm=/home/philipp/data/snap_files/my_genome.hmm #SNAP HMM file```

To base the predictions in the second MAKER run only on SNAP  remove the filepaths to the protein and est evidence or set the flags for `est2genome=0` and `protein2genome=0`.

Now in the folder for your predictions execute MAKER for the second time: 
	
	$ maker
	

# Step 4

After MAKER is finished repeat Step 2 to create another set of hmms for SNAP. This can be done several times, however beware because SNAP can also be overtrained.

# Step 5

Repeat Step 3 with the new hmm file created in Step 4.


# Step 6

We will now train the second gene predictor [AUGUSTUS](http://augustus.gobics.de).

First follow Step 2 till you have the `export.ann` and `export.dna` files. AUGUSTUS uses the GeneBank [GBK](https://www.ncbi.nlm.nih.gov/Sitemap/samplerecord.html) format file as input. Unfortunately this is difficult to get starting from the ZFF files which SNAP creates. Thankfully, Jason Stajich has created a nice Perl script which converts SNAP ZFF files to GBK files. It is available on his GitHub page: 
`https://github.com/hyphaltip/genome-scripts/blob/master/gene_prediction/zff2augustus_gbk.pl`

The script needs to be run in the directory containing the `export.ann` and `export.dna` files.

	$ zff2augustus_gbk.pl > augustus.gbk
	
Like in many machine learning approaches, we will split the now created `augustus.gbk` file into a training and a test set:

	$ perl randomSplit.pl augustus.gbk 100

`randomSplit.pl` comes with AUGUSTUS and is located in the scripts folder.

We now have to create a new species for our AUGUSTUS training:

	$ new_species.pl --species=my_species
	
this script is also located in the AUGUSTUS/scripts directory.

Lets now train AUGUSTUS with the training set file, evaluate the training and save the output of this testto a file for later reference:

	$ etraining --species=my_species augustus.gbk.train
	$ augustus --species=my_species augustus.gbk.test | tee first_training.out

Once this is done, we are ready to improve prediction parameters of the models using the `optimize_augustus.pl` script again located in the AUGUSTUS/scripts directory: 

	$ optimize_augustus.pl --species=my_species augustus.gbk.train 
	
After this is done (this will take some time!) we can retrain and test AUGUSTUS with the optimized paramters and compare the results to the first run:

	$ etraining --species=my_species augustus.gbk.train
	$ augustus --species=my_species augustus.gbk.test | tee second_training.out
	
	
Look [here](https://vcru.wisc.edu/simonlab/bioinformatics/programs/augustus/docs/tutorial2015/training.html) on how to interpret training results.

# Step 7

We are now ready to rerun MAKER including evidence gene predictions from AUGUSTUS.

For this change the following lines in your `maker_opts.ctl` file to include the species name of your AUGUSTUS species.

`augustus_species=my_species #Augustus gene prediction species model`

you may also set the switch to keep all predictions in the same file:

`keep_preds=1 #Concordance threshold to add unsupported gene prediction (bound by 0 and 1)`

Now rerun MAKER:

	$ maker

# Step 8

It is now time to train the last gene predictor [GeneMark](http://opal.biology.gatech.edu/genemark/).
GeneMark is self-training and all you need is your genome assembly file. Execute the following command:

	$ gmes_petap.pl -ES -fungus -cores 10 -sequence /home/data/my_assembly.fasta
	
This is just an example and you have to make sure to set the correct parameters suitable for your organism and computer. Please refer to the manual of GeneMark for reference.

GeneMark will create a `gmhmm.mod` file which needs to be added to the `maker_opts.ctl` control file by changing the according line to something like this:

`gmhmm=/home/philipp/data/gmhmm.mod #GeneMark HMM file`

# Step 9

Run MAKER for the last time, now with evidence from SNAP, AUGUSTUS and GeneMark:

	$ maker


Hopefully everything has worked as it should and you will see the *Maker is now finished!!!* message on screen.

![MAKER finished](https://github.com/reslp/blog/raw/master/images/maker_fin.png)

# Further reading:

[A nice MAKER Tutorial @ The MAKER Wiki](http://weatherby.genetics.utah.edu/MAKER/wiki/index.php/MAKER_Tutorial_for_GMOD_Online_Training_2014)

[The MAKER Manual](http://weatherby.genetics.utah.edu/MAKER/wiki/index.php/Main_Page)

[The MAKER Google group](https://groups.google.com/forum/#!forum/maker-devel)

 