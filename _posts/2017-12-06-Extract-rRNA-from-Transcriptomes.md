---
layout: post
title: How-to extract rRNA from assembled (meta-)transcriptomes
---
The title says it all: Here I describe how I extracted rRNA from lichen (meta-)transcriptomes.

# Introduction

I recently had to extract rRNA from a large number of lichen metatranscriptomes. There are multiple ways to approach this but I decided to use RNAmmer. I found the approach RNAmmer takes very attractive because searches for rRNAs based on a database of HMMs of known rRNA genes rather than using a blast best hit approach. [RNAmmer](http://www.cbs.dtu.dk/cgi-bin/sw_request?rnammer) was originally designed to predict rRNA genes in genomes, however the [Trinotate](https://trinotate.github.io) pipeline (for annotating transcriptomes) provides a nice wrapper script to adopt RNAmmer for transcriptomes. We were also interested in the taxonomic composition of the extracted rRNA so it was necessary to get lineage information for the rRNA transcripts.
Here is how to do all this:

1. First download and install [Trinotate](https://trinotate.github.io). You will not need to install additional software such as SignalP or TransDecoder, but you will need to install [RNAmmer](http://www.cbs.dtu.dk/cgi-bin/sw_request?rnammer).
2. Download and Install [RNAmmer](http://www.cbs.dtu.dk/cgi-bin/sw_request?rnammer) and follow the instructions on the Trinotate website on how to modify RNAmmer.

*Note: Follow the instructions on how to install RNAmmer on the Trinotate website carefully, otherwise you may run into problems.*

3. Once you have installed Trinotate and RNAmmer you can run the wrapper script on your transcriptome. The script you want to use is called *RnammerTranscriptome.pl*. It is located in yourtrinotatedir/utils/rnammer_support/RnammerTranscriptome.pl. You may run it like this:

```
/usr/local/src/Trinotate-Trinotate-v3.1.0/util/rnammer_support/RnammerTranscriptome.pl --transcriptome ../../sticta/Sticta_canariensis/trinity_cyano/Trinity.fasta --path_to_rnammer /usr/local/src/rnammer-1.2/rnammer
```

*Note that the PATH to rnammer has to be absolute!*

4. This will create several files. The discovered rRNAs are in `Trinity.fasta.rnammer.gff`. This file is a standard GFF3 file (look [here](http://www.ensembl.org/info/website/upload/gff3.html) if you would like to know more about GFF3 files). This is how a typical rnammer output looks like:

![RNAhmmer](https://github.com/reslp/blog/raw/master/images/rnammer-output.png)

5. I created an intermediate file with the sequence IDs in FASTA format derived from the GFF file created in the previous step. This is because I have a python script ready which selects only specific sequences based on their names, which we will be using in the next step. 
```
cat Trinity.fasta.rnammer.gff | awk 'BEGIN { FS="\t" } {print ">"$1}' | uniq > rRNA_transcript_ids.txt
```

6. The script I will be using is called `select_transcripts.py` and you can get it from my [Github](https://github.com/reslp/genomics) genomics repository. It is executed like this: 
```
select_transcripts.py Trinity.fasta rRNA_transcript_ids.txt normal > rRNA_transcripts.fasta
``

7. We can now blastn the selected sequences to get the lineage information of the closest hit from the NCBI database. I created a local copy of the nt database on your server, however it is also possible to blast the sequences remotely.

```
blastn -db nt -query rRNA_transcripts.fasta -out taxids.txt -outfmt "6 qseqid staxids sseqid pident qlen length mismatch gapope evalue bitscore" -max_target_seqs 1
```

*Note: You can use the `-remote` flag to blast against the online version NCBI database directly, however you will nead a blast version >2.6 for this to work.*

I used tabular output format and I only keep the best hit. Keeping only single hits can be problematic, especially when a sequence has hits in multiple unrelated taxa. It would also be possible (and probably a better idea to load blast results into MEGAN and create some kind of consensus taxonomic assignment). In my case I was interested to assign rRNA to either bacteria, archaea or eukarya. Therefore I decided a single hit should be fine.

8. Blast returns taxonids which can be associated with the lineage information contained in the NCBI databases. I have created a small R script using the taxize package to retrieve this information. It is calles `retrieve_lineage_from_taxid.R`and you can get it from my genomics [GitHub](https://github.com/reslp/genomics) repository. You can use it like this: 

```
retrieve_lineage_from_taxid.R taxids.txt > taxid_and_lineage_info.txt
```
The output may look something like this:
![RNAhmmer](https://github.com/reslp/blog/raw/master/images/final_output_rrna.png)

This is again blast tabular output with added a seperate column with lineage information.


# Final remarks

Several steps of this workflow can be improved and/or combined. For example one could use AWK to extract the first column of the GFF3 output file to directly select sequences from the assembly. I just like to keep different steps seperate to keep analyses steps more modular. 
Another possible improvement would be to use GNU [parallel](https://www.gnu.org/software/parallel/) with blast to speed up the blasting procedure. I actually did this for my analysis, but for this post I wanted to keep it simple.
Finally, I have also created a simple shell script which summarizes all the above commands to extract bacterial, archaeal and eukaryotic rRNA in one go. You can get it [here](https://github.com/reslp/genomics/blob/master/extract_rrna.sh).

Hope this is useful for someone ;)


