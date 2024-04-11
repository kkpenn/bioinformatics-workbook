---
title: "Using PLINK to calculate linkage disequilibrium"
layout: single
author: Kayla Pennerman
author_profile: true
header:
  overlay_color: "444444"
  overlay_image: /assets/images/dna.jpg
---



# Using Plink to calculate linkage disequilibrium

## Introduction

[`PLINK`](https://github.com/chrchang/plink-ng/) is a free and open-soure tool that is able to efficiently perform whole genome analyses ([Chang et al., 2015](https://academic.oup.com/gigascience/article/4/1/s13742-015-0047-8/2707533?login=false)). One of the many operations one can perform with PLINK is to calculate linkage disequilibrium, an indicator of nonrandom associations of alleles at different loci. This can be useful in determining a population's recombination history, allelic ages, mutation rates, etc ([Slatkin 2008](https://www.nature.com/articles/nrg2361)).

## Dataset

For this tutorial, we will use a variant call format (VCF) file containing allelic data for _Saccharomyces eubayanus_ (https://figshare.com/articles/dataset/VCF_files_of_industrial_yeasts/12673250). This dataset is from Walter Pfliegler and  Hanna Rácz and is the result of GATK genomic analyses. Sequencing reads from five individuals were mapped to the _Saccharomyces cerevisiae_ S288C reference genome and single nucleotide polymorphisms (SNPs) were called and filtered. After decompressing the downloaded folder, you will find a VCF file titled "eubayanus_filtered.vcf" (to uncompress: unrar x <filename>).

[`The VCF version 4.2 specification`](https://samtools.github.io/hts-specs/VCFv4.2.pdf) explains how VCF files are formatted. If you need to learn how to generate your own VCFs, [`check out other tutorials`](https://bioinformaticsworkbook.org/dataAnalysis/VariantCalling/variant-calling-index#gsc.tab=0) in this workbook.

## Required software

Python version 3 (the re module is built-in; https://www.python.org)
VCFtools version 0.1.16 (https://vcftools.github.io/index.html)
PLINK version 1.9 (https://www.cog-genomics.org/plink/)
R version 4 (the ggplot library is built-in; https://www.r-project.org/)

Software installation/set-up may vary depending on your environment.


## Workflow

### Quality check the file

A simple script used in the command line can be used to check that each locus only describes a single nucleotide variation. In our VCF file, there are variations that are ambiguous or involve more than just a single nucleotide change. We'll just remove those with a quick Python script, ending up removing 370 loci and giving each locus a name:

```
import re

def snp_only(i_v, o_v):
    '''make new file with only SNPs and add IDs to SNP loci'''
    with open(i_v, 'r') as I_file:
        with open(o_v, 'w') as O_file:
            for read_line in I_file.readlines():
                if re.search('^#', read_line):
                    O_file.write(read_line)
                else:
                    split_line = read_line.split()
                    if read_line[-1] == '\n':
                        split_line[-1] += '\n'
                    if len(split_line[3]) > 1 or len(split_line[4]) > 1:
                        continue
                    split_line[2] = split_line[0]+'_'+split_line[1]
                    O_file.write('\t'.join(split_line))

snp_only('eubayanus_filtered.vcf', 'eubayanus_snps.vcf')
```

It is possible that individuals aren't genotyped at different loci. To check that, run the following:

```
#depending on your environment, you may need to use "./plink" instead of "plink"
plink --missing --allow-extra-chr --vcf eubayanus_snps.vcf
```

PLINK expects human chromosome IDS, but the flag "--allow-extra-chr" permits other names for genomic segments. The above command yields two files: plink.lmiss and plink.imiss. The former lists missing genotypic and phenotypic data for each SNP locus. The latter helpfully summarizes that information into a table:

Table 1: Missing data in the VCF file

|     FID     |   ID   | MISS_PHENO | N_MISS | N_GENO | F_MISS |
|-------------|--------|------------|--------|--------|--------|
|    W34-70   | A1B11  |      Y     |  3149  | 69617  |0.04523 |
|    W34-70   | A1     |      Y     |  2570  | 69617  |0.03692 |
|    W34-70   | A2     |      Y     |  2869  | 69617  |0.04121 |
|    W34-70   | Fay    |      Y     | 15104  | 69617  |0.217   |
|Weihenstephan| 34-70  |      Y     |  1070  | 69617  |0.01537 |

Hmm, W34-70 Fay is missing a lot more data than the others. Let's remove those loci and get to an F_MISS rate below 10% for all individuals. Note that we use the flag "--max-missing" below with a value of 0.9 to request 10% or less missing data for each locus. Other flags exclude indels ("--remove-indels"), set the minimum quality score ("--minQ") to 30 and the minimum read depth ("--min-meanDP") to 10. The flag "--recode" indicates that the output should be in a VCF file, which we pipe out to the stdout ("--stdout") and capture in a new file.

```
vcftools --vcf eubayanus_snps.vcf --remove-indels --max-missing 0.9 --minQ 30 --min-meanDP 10 --recode --stdout > eubayanus_nomissing.vcf
```

Now there are 53,524 SNP loci with no missing data. (How can you check that? And why wouldn't there be any missing data? Hint: think of how many individuals are in the dataset.)

Pruning is best practice for getting a subset of SNP loci that are not highly correlated with each other. You can perform it with the following code:

```
plink --allow-extra-chr --vcf eubayanus_nomissing.vcf --indep-pairwise 5 5 0.5
plink --allow-extra-chr --vcf eubayanus_nomissing.vcf --extract plink.prune.in --make-bed --out pruned
plink --allow-extra-chr --bfile pruned --recode vcf --out eubayanus_pruned
```

The flag "--indep-pairwise" requires a window size in number of loci (or in kilobases if you use the "kb" modifier), the number of loci to shift the window by and a squared correlation threshold. Feel free to try different parameters to see how they affect the outcomes.

The first command outputs two files: plink.prune.in and plink.prune.out, both of which are lists of the IDs of SNP loci (the IDs we added with the Python script earlier) that are included or excludeded, respectively. With the second command, plink.prune.in is used to create a binary file with just the loci listed in the file, then the third command generates a VCF file from the binary file.

We end up with just 1,132 SNPs

### Calculate LDs

Now, we're ready for the main event! It's just one line of code:

```
plink --allow-extra-chr --vcf eubayanus_pruned.vcf --r2 --out eubayanus_pruned --ld-window 10000 --ld-window-r2 0
```

Here, we ask PLINK to perform pairwise comparisons of the SNP loci to calculate squared correlations ("--r2" flag). We stipulate that loci should be within 10,000 loci of each other (not a problem for only 1,132 loci!) with the "--ld-window" flag and that the minimum squared correlation that should be in the output is 0 with the "--ld-window-r2". A third flag, "--ld-window", can set the parameter to limit the distance in kilobases between SNP loci. Default parameters can be viewed [`here`]`(https://www.cog-genomics.org/plink/1.9/ld).

Since PLINK is so quick, let's also calculate run the command on the unpruned SNP set for comparison:

```
plink --allow-extra-chr --vcf eubayanus_nomissing.vcf --r2 --out eubayanus_full --ld-window 10000 --ld-window-r2 0
```

### Visualize the LD decay

The resulting "eubayanus_pruned.ld" and "eubayanus_full.ld" files can be directly used to generate LD decay plots in R:

```
library(ggplot2)

#import the data
ldf <- read.table("eubayanus_full.ld", header=T)
ldp <- read.table("eubayanus_pruned.ld", header=T)

#calculate average correlations
ldf$Distance <- abs(ldf$BP_B-ldf$BP_A)/1000
ldf <- ldf[!(ldf$Distance > 500),]
lw_ldf <- loess(R2 ~ Distance, data=ldf)
j_ldf <- order(ldf$Distance)
fit_dataf <- data.frame(ldf$Distance[j_ldf], lw_ldf$fitted[j_ldf])

ldp$Distance <- abs(ldp$BP_B-ldp$BP_A)/1000
ldp <- ldp[!(ldp$Distance > 500),]
lw_ldp <- loess(R2 ~ Distance, data=ldp)
j_ldp <- order(ldp$Distance)
fit_datap <- data.frame(ldp$Distance[j_ldp], lw_ldp$fitted[j_ldp])

#plot the fitted data
decay_plot <- ggplot() + geom_point(size=0.02) + 
  geom_point(data=fit_dataf,
             aes(x=ldf.Distance.j_ldf., y=lw_ldf.fitted.j_ldf.),
             size=0.02, color='black') +
  geom_point(data=fit_datap,
             aes(x=ldp.Distance.j_ldp., y=lw_ldp.fitted.j_ldp.),
             size=0.02, color='red') +
  scale_x_continuous(expand=c(0,0), name="Distance (kbp)") + 
  scale_y_continuous(expand=c(0,0), name=bquote(r^2)) + theme_bw()
decay_plot
ggsave('eubayanes_ld.png')
```

And we're done! You should have a graph that looks just like this:

![graph](assets/eubayanus_ld.png)

Fig 1: LD decay of _S_. _eubayanes_ with pruned (red) and not pruned (black) SNP data