# MutationalPatterns

The MutationalPatterns R package provides a comprehensive set of flexible functions for easy finding and plotting of mutational patterns in base substitution catalogues.

NEW RELEASE: 
* Faster vcf file loading
* Automatic check and exclusion of positions in vcf with indels and/or multiple alternative alleles 
* Default plotting colours to standard in the mutational signatures field

NEW FUNCTIONALITIES
* Function to make chromosome names uniform according to e.g. UCSC standard
* Transcriptional strand bias analysis
* Signature extraction (NMF) with transcriptional strand information
* Enrichment/depletion test for genomic annotations

Please give credit and cite MutationalPatterns R Package when you use it for your data analysis. For information on how to cite this package in your publication execute:

PAPER ON bioRxiv

  ```{r}
  citation("MutationalPatterns")
  ```


## Getting started

### Installation

This package is dependent on R version 3.3.1

Install and load devtools package

  ```{r}
  install.packages("devtools")
  library(devtools)
  ```
Install and load MutationalPatterns package

  ```{r}
  options(unzip = 'internal')
  install_github("CuppenResearch/MutationalPatterns")
  library(MutationalPatterns)
  ```

### Reference genome

1. List all available reference genomes (BSgenome)

  ```{r}
  library(BSgenome)
  available.genomes()
  ```
2. Download and load your reference genome of interest

  ```{r}
  ref_genome = "BSgenome.Hsapiens.UCSC.hg19"
  source("http://bioconductor.org/biocLite.R")
  biocLite(ref_genome)
  library(ref_genome, character.only = T)
  ```
  
### Load base substitution data

This package is for the analysis of patterns in SNV data only, therefore the vcf files should not contain indel positions.

Find package base substitution example/test data
  ```{r}
  vcf_files = list.files(system.file("extdata", package="MutationalPatterns"), full.names = T)
  ```

Load a single vcf file
  ```{r}
  vcf = read_vcf(vcf_files[1], "sample1")
  ```

Load a list of vcf files
  ```{r}
  sample_names = c("colon1", "colon2", "colon3", "intestine1", "intestine2", "intestine3", "liver1", "liver2", "liver3")
  vcfs = read_vcf(vcf_files, sample_names, genome = "hg19")
  ```

Include relevant metadata in your analysis, e.g. donor id, cell type, age, tissue type, mutant or wild type
  ```{r}
  tissue = c("colon", "colon", "colon", "intestine", "intestine", "intestine", "liver", "liver", "liver")
  ```

### Make chromosome names uniform

Check if chromosome names in vcf(s) and reference genome are the same
  ```{r}
  all(seqlevels(vcfs[[1]]) %in% seqlevels(get(ref_genome)))
  ```

If not, rename the seqlevels to the UCSC standard
  ```{r}
  vcfs = lapply(vcfs, function(x) rename_chrom(x))
  ```
  
Select autosomal chromosomes
  ```{r}
  auto = extractSeqlevelsByGroup(species="Homo_sapiens", style="UCSC", group="auto")
  vcfs = lapply(vcfs, function(x) keepSeqlevels(x, auto))
  ```

##  Analyses

### Mutation types

Retrieve base substitutions from vcf object as "REF>ALT"
  ```{r}
  get_muts(vcfs[[1]])
  ```
  
Retrieve base substitutions from vcf and convert to the 6 types of base substitution types that are distinguished by convention: C>A, C>G, C>T, T>A, T>C, T>G. For example, if the reference allele is G and the alternative allele is T (G>T), this functions returns the G:C>T:A mutation as a C>A mutation.
  ```{r}
  get_types(vcfs[[1]])
  ```
  
Retrieve the context (1 base upstream and 1 base downstream) of the positions in the vcf object from the reference genome.
  ```{r}
  get_mut_context(vcfs[[1]], ref_genome)
  ```

Retrieve the types and context of the base substitution types for all positions in the vcf object. For the base substitutions that are converted to the conventional base substitution types, the reverse complement of the context is returned.
  ```{r}
  get_type_context(vcfs[[1]], ref_genome)
  ```

Count mutation type occurences for one vcf object
  ```{r}
  type_occurences = mut_type_occurences(vcfs[1], ref_genome)
  ```

Count mutation type occurences for all samples in a list of vcf objects
  ```{r}
  type_occurences = mut_type_occurences(vcfs, ref_genome)
  ```

### Mutation spectrum

Plot mutation spectrum over all samples. Plots the mean relative contribution of each of the 6 base substitution types. Error bars indicate standard deviation over all samples. The n indicates the total number of mutations in the set.
  ```{r}
  plot_spectrum(type_occurences)
  ```

Plot mutation spectrum with distinction between C>T at CpG sites
  ```{r}
  plot_spectrum(type_occurences, CT = T)
  ```

Plot spectrum without legend
  ```{r}
  plot_spectrum(type_occurences, CT = T, legend = F)
  ```

  ![spectra1](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/spectra1.png)


Plot spectrum for each tissue separately
  ```{r}
  plot_spectrum(type_occurences, by = tissue, CT = T)
  ```

Specify 7 colors for spectrum plotting
  ```{r}
  my_colors = c("pink", "orange", "blue", "lightblue", "green", "red", "purple")
  plot_spectrum(type_occurences, CT = T, legend = T, colors = my_colors)
  ```
  
  ![spectra2](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/spectra2.png)

### 96 Mutation Profile

Make 96 trinucleodide mutation count matrix
  ```{r}
  mut_matrix = mut_matrix(vcf_list = vcfs, ref_genome = ref_genome)
  ```

Plot 96 profile of three samples
  ```{r}
  plot_96_profile(mut_matrix[,c(1,4,7)])
  ```
  ![96_mutation_profile](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/96_profile.png)

### Extract Signatures

Estimate optimal rank for NMF mutation matrix decomposition

  ```{r}
  estimate_rank(mut_matrix, rank_range = 2:5, nrun = 50)
  ```

  ![estim_rank](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/estim_rank.png)

Extract and plot 3 signatures

  ```{r}
  nmf_res = extract_signatures(mut_matrix, rank = 3)
  # provide signature names (optional)
  colnames(nmf_res$signatures) = c("Signature A", "Signature B" , "Signature C")
  # plot signatures
  plot_96_profile(nmf_res$signatures)
  ```

  ![signatures](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/signatures.png)

Plot signature contribution

  ```{r}
  # provide signature names (optional)
  rownames(nmf_res$contribution) = c("Signature A", "Signature B" , "Signature C")
  # plot relative signature contribution
  plot_contribution(nmf_res$contribution, nmf_res$signature, mode = "relative")
  # plot absolute signature contribution
  plot_contribution(nmf_res$contribution, nmf_res$signature, mode = "absolute")
  ```

  ![contribution1](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/contribution1.png)
  
  ```{r}
  # plot contribution of signatures for subset of samples with index parameter
  plot_contribution(nmf_res$contribution, nmf_res$signature, mode = "absolute", index = c(1,2))
  # flip X and Y coordinates
  plot_contribution(nmf_res$contribution, nmf_res$signature, mode = "absolute", coord_flip = T)
  ```
  
  ![contribution2](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/contribution2.png)

Compare reconstructed mutation profile with original mutation profile

  ```{r}
  plot_compare_profiles(mut_matrix[,1], nmf_res$reconstructed[,1], profile_names = c("Original", "Reconstructed"))
  ```

  ![originalVSreconstructed](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/original_VS_reconstructed.png)

### Fit 96 mutation profiles to known signatures  

Download signatures from pan-cancer study Alexandrov et al.
  
  ```{r}
  url = "http://cancer.sanger.ac.uk/cancergenome/assets/signatures_probabilities.txt"
  cancer_signatures = read.table(url, sep = "\t", header = T)
  # reorder (to make the order of the trinucleotide changes the same)
  cancer_signatures = cancer_signatures[order(cancer_signatures[,1]),]
  # only signatures in matrix
  cancer_signatures = as.matrix(cancer_signatures[,4:33])
  ```

Fit mutation matrix to cancer signatures. This function finds the optimal linear combination of mutation signatures that most closely reconstructs the mutation matrix by solving nonnegative least-squares constraints problem

  ```{r}
  fit_res = fit_to_signatures(mut_matrix, cancer_signatures)
  # select signatures with some contribution
  select = which(rowSums(fit_res$contribution) > 0)
  # plot contribution
  plot_contribution(fit_res$contribution[select,], cancer_signatures[,select], coord_flip = F, mode = "absolute")

  ```

  ![signatures](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/contribution_cancer_sigs.png)

Compare reconstructed mutation profile of sample 1 using cancer signatures with original profile

  ```{r}
  plot_compare_profiles(mut_matrix[,1], fit_res$reconstructed[,1], profile_names = c("Original", "Reconstructed \n cancer signatures"))
  ```

  ![contribution](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/original_VS_reconstructed_cancer_sigs.png)


### Transcriptional strand bias analysis

For the mutations within genes it can be determined whether the mutation is on the transcribed or non-transcribed strand, which is interesting to study involvement of transcription-coupled repair. To this end, it is determined whether the "C" or "T" base (since by convention we regard base substitutions as C>X or T>X) are on the same strand as the gene definition. Base substitions on the same strand as the gene definitions are considered "untranscribed", and on the opposite strand of gene bodies as transcribed, since the gene definitions report the coding or sense strand, which is untranscribed. No strand information is reported for base substitution that overlap with more than one gene body.

Find gene definitions for your reference genome.

  ```{r}
  # get knowngenes table from UCSC for hg19
  biocLite("TxDb.Hsapiens.UCSC.hg19.knownGene")
  library("TxDb.Hsapiens.UCSC.hg19.knownGene")
  genes_hg19 = genes(TxDb.Hsapiens.UCSC.hg19.knownGene)
  ```
Make mutation count matrix with transcriptional strand information (96 trinucleotides * 2 strands = 192 features). NB: only those mutations that are located within gene bodies are counted.

  ```{r}
  mut_mat_s = mut_matrix_stranded(vcfs, ref_genome, genes_hg19)
  ```
  
Perform strand bias analysis

  ```{r}
  strand_counts = strand_occurences(mut_mat_s, by=tissue)
  strand_plot = plot_strand(strand_counts, mode = "relative")
  ```

Perform poisson test for strand asymmetry significance testing

  ```{r}
  strand_bias = strand_bias_test(strand_counts)
  strand_bias_plot = plot_strand_bias(strand_bias)
  ```
  
  ![strand](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/strand.png)  

  
## Extract signatures with strand bias


  ```{r}
  # extract 2 signatures
  nmf_res_strand = extract_signatures(mut_mat_s, rank = 2)
  
  # provide signature names (optional)
  colnames(nmf_res_strand$signatures) = c("Signature A", "Signature B")
  # plot signatures with 192 features
  plot_192_profile(nmf_res_strand$signatures)
  
  # provide signature names (optional)
  rownames(nmf_res_strand$contribution) = c("Signature A", "Signature B")
  # plot signature contribution
  plot_contribution(nmf_res_strand$contribution, nmf_res_strand$signatures, coord_flip = T, mode = "absolute")
  ```
  

  ![signatures_strand](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/signatures_strand.png)  







### Rainfall plot

A rainfall plot visualizes mutation types and intermutation distance. Rainfall plots can be used to visualize the distribution of mutations along the genome or a subset of chromosomes. The y-axis corresponds to the distance of a mutation with the previous mutation and is log10 transformed. Drop-downs from the plots indicate clusters or "hotspots" of mutations.

Make rainfall plot of sample 1 over all autosomal chromosomes
  ```{r}
  # define autosomal chromosomes
  chromosomes = seqnames(get(ref_genome))[1:22]
  # make rainfall plot
  plot_rainfall(vcfs[[1]], title = names(vcfs[1]), ref_genome = ref_genome, chromosomes = chromosomes, cex = 1.5)

  ```

  ![rainfall1](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/rainfall1.png)
  
Make rainfall plot of sample 1 over chromosome 1

  ```{r}
  chromosomes = seqnames(get(ref_genome))[1]
  plot_rainfall(vcfs[[1]], title = names(vcfs[1]), ref_genome = ref_genome, chromosomes = chromosomes[1], cex = 2)
  ```
  ![rainfall2](https://github.com/CuppenResearch/MutationalPatterns/blob/develop/images/rainfall2.png)