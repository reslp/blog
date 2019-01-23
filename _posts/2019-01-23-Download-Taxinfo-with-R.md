---
layout: post
title: Download lineage information for species names and taxids with R
---
# Introduction

I have always struggled with how I can efficiently get the lineage information for an NCBI taxid. I have downloaded the complete taxonomy files from NCBI and tried to parse it locally. I can tell you this has been painful. Recently I had to retrieve lineage information for many thousands of species names and taxids so I have invested some time into searching for a better solution to this problem. Although there are several R package accessing the NCBI API no single package could give me the desired output. I ended up using three R packages [myTAI](https://cran.r-project.org/package=myTAI) and [taxize](https://cran.r-project.org/web/packages/taxize/index.html). Here is the workflow:

# Prerequisites

First we need to install and load the required packages

```
install.packages(c("taxize","myTAI","plyr")
library(taxize)
library(myTAI)
library(plyr)

```

# Download taxon summary

Next we have to download all the taxonomy summary information for each element in a given vector with taxids. This will retrieve a summary dataframe containing the taxon name, its rank and taxonid.
It is only necessary to do this if you start from taxids. If you have species names you can skip this step.

```
taxon_summary <- ncbi_get_taxon_summary(id = taxids)
```

**Note:** You may consider getting an ENTREZ key to be able to send more requests to the NCBI API. Look [here] (https://ncbiinsights.ncbi.nlm.nih.gov/2017/11/02/new-api-keys-for-the-e-utilities/) on how to do this. You can add the key by supplying the argument `key="<your_key>` to the `ncbi_get_taxon_summary` function.

# Get complete lineage information

We can now use this summary dataframe to retrieve the complete lineage information using *myTAI*. Here we create an empty list which will contain the lineage information for each individual query. For each query we create a dataframe with the lineage information. It is important that this dataframe has named columns, since we need them later when we combine the list into a single dataframe. I also added a `Sys.sleep(0)` command since NCBI restricts access to its API when it receives to my requests.
```
df_list <- list()
for (i in 1:nrow(taxon_summary)){
tax  <- taxonomy(organism = taxon_summary[i,]$name, db = "ncbi",output = "classification")
df <- data.frame(lapply(tax$name, function(x) data.frame(x)))
colnames(df) <- tax$rank
df_list[[i]] <- df
Sys.sleep(0.5)
}
```

# Combining the output to a single dataframe

Once this is done we can combine the list of dataframes into a single dataframes. Missing values will be filled with `<NA>` automatically, all with the magic of *dplyr*.
```
combined_df <- do.call(rbind.fill, df_list)
```

# The whole script

Here are all the different parts combined while adding some example taxonids as a test:

```
install.packages(c("taxize","myTAI","plyr")
library(taxize)
library(myTAI)
library(plyr)

taxids <- c(644223,1099808,169507,324739,324740)

taxon_summary <- ncbi_get_taxon_summary(id = taxids)

df_list <- list()
for (i in 1:nrow(taxon_summary)){
tax  <- taxonomy(organism = taxon_summary[i,]$name, db = "ncbi",output = "classification")
df <- data.frame(lapply(tax$name, function(x) data.frame(x)))
colnames(df) <- tax$rank
df_list[[i]] <- df
Sys.sleep(0.5)
}

combined_df <- do.call(rbind.fill, df_list)
```

For this example the final dataframe `combined_df` looks like this:

```
 no rank superkingdom kingdom subkingdom     phylum        subphylum           class             order   
1 cellular organisms    Eukaryota   Fungi    Dikarya Ascomycota Saccharomycotina Saccharomycetes Saccharomycetales
2 cellular organisms    Eukaryota   Fungi    Dikarya Ascomycota Saccharomycotina Saccharomycetes Saccharomycetales
3 cellular organisms    Eukaryota   Fungi    Dikarya Ascomycota Saccharomycotina Saccharomycetes Saccharomycetales
4 cellular organisms    Eukaryota   Fungi    Dikarya Ascomycota Saccharomycotina Saccharomycetes Saccharomycetales
5 cellular organisms    Eukaryota   Fungi    Dikarya Ascomycota Saccharomycotina Saccharomycetes Saccharomycetales
            family         genus                     species
1 Phaffomycetaceae  Komagataella        Komagataella phaffii
2 Phaffomycetaceae  Komagataella         Komagataella populi
3 Phaffomycetaceae  Komagataella Komagataella pseudopastoris
4       Pichiaceae Kregervanrija    Kregervanrija delftensis
5       Pichiaceae Kregervanrija       Kregervanrija fluxuum
```

Here is also my Rsession information:

```
> sessionInfo()
R version 3.3.3 (2017-03-06)
Platform: x86_64-apple-darwin13.4.0 (64-bit)
Running under: macOS  10.14

locale:
[1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8

attached base packages:
[1] stats     graphics  grDevices utils     datasets  methods   base     

other attached packages:
[1] plyr_1.8.4           taxize_0.9.4         myTAI_0.8.0          biomartr_0.9.9000    BiocInstaller_1.24.0
[6] bindrcpp_0.2.2       biomaRt_2.30.0      

loaded via a namespace (and not attached):
 [1] Rcpp_1.0.0           ape_5.0              lattice_0.20-34      Biostrings_2.42.1    zoo_1.8-1           
 [6] utf8_1.1.3           assertthat_0.2.0     digest_0.6.15        foreach_1.4.4        R6_2.2.2            
[11] stats4_3.3.3         RSQLite_2.0          httr_1.3.1           pillar_1.2.1         zlibbioc_1.20.0     
[16] rlang_0.3.1          curl_3.1             rstudioapi_0.7       data.table_1.10.4-3  blob_1.1.0          
[21] S4Vectors_0.12.2     urltools_1.6.0       devtools_1.13.4      downloader_0.4       readr_1.3.1         
[26] stringr_1.3.0        RCurl_1.95-4.10      bit_1.1-12           triebeard_0.3.0      pkgconfig_2.0.1     
[31] BiocGenerics_0.20.0  tidyselect_0.2.5     tibble_1.4.2         httpcode_0.2.0       IRanges_2.8.2       
[36] codetools_0.2-15     XML_3.98-1.10        reshape_0.8.8        crayon_1.3.4         dplyr_0.7.8         
[41] withr_2.1.1          bitops_1.0-6         crul_0.7.0           grid_3.3.3           nlme_3.1-131        
[46] jsonlite_1.5         DBI_0.7              git2r_0.21.0         magrittr_1.5         cli_1.0.0           
[51] stringi_1.1.6        XVector_0.14.1       reshape2_1.4.3       xml2_1.1.1           iterators_1.0.10    
[56] tools_3.3.3          bold_0.8.6           bit64_0.9-7          Biobase_2.34.0       glue_1.3.0          
[61] purrr_0.2.5          hms_0.4.2            parallel_3.3.3       yaml_2.2.0           AnnotationDbi_1.36.2
[66] memoise_1.1.0        knitr_1.20           bindr_0.1.1 
```


# Final remarks

Of course this simple script can be extended in many ways to incorporate exception handling, adding additional columns to the final dataframe and many more. I still hope that this will be useful to some people. 



