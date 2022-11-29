# Ribosome footprints quality control {#rpfs-qc}


In this chapter, we assessed the quality of the ribosome footprints (RPFs) in several aspects by using Bioconductor’s _ribosomeProfilingQC_ package and the in-hoouse ad-hoc functions. The aspects include:  

1. size of RPFs: calculating the size distribution of the RPFs, confirming that most RPFs are congruent with the actual size of ribosomes (26 - 29 nt)
2. TSS off-set enrichment: visualizing the distance from 5' end of reads to the start codon to decide the best off-set for p-sites
3. p-sites and reading frames: estimating p-sites coordinates of RPFs of the dominant length (26 - 29 nt) and ensuring p-sites are in-frame around the start codon
4. trinucleotide periodicity on transcripts: constructing meta-gene p-sites coverage plot, colored by reading frames, to show the trinucleotide footprint periodicity from 5' UTR, CDS, to 3' UTR

After ensuring the quality of RPFs and p-sites, we profiled PRFs by counting p-sites on different genomic features including 5' UTR, CDS, and 3' UTR.

The main script performed the RPFs QC is [here](https://github.com/FredHutch/DUX4-IFNg-ribosome-footprints/scripts/020-use_riboProfilingQC.R). Note that we do not include BAM files in our repository. Since all code chunks in this chapter involve the BAM files and therefore would not be evaluated. This means the figures in this section were pre-generated.

## Preparation
The code chunk below loads the libraries:

```r
library(ribosomeProfilingQC)
library(tidyverse)
library(DESeq2)
library(Rsamtools)
library(GenomicFeatures)
library(GenomicAlignments)
library(hg38.HomoSapiens.Gencode.v35)
txdb <- hg38.HomoSapiens.Gencode.v35
library(BSgenome.Hsapiens.UCSC.hg38)
genome <- BSgenome.Hsapiens.UCSC.hg38
library(BiocParallel)
bp_param=MulticoreParam(workers = 4L)
register(bp_param, default=TRUE)

pkg_dir <- "/fh/fast/tapscott_s/CompBio/Ribo-seq/hg38.DUX4.IFN.ribofootprint.2"
scratch_dir <- "/fh/scratch/delete90/tapscott_s/hg38.DUX4.IFN.ribofootprint.R1"
fig_dir <- file.path(pkg_dir, "figures", "QC")
source(file.path(pkg_dir, "scripts", "tools.R"))
source(file.path(pkg_dir, "scripts", "fork_readsEndPlot.R"))
```

Building sample information:

```r
bam_dir <- file.path(scratch_dir, "bam", "merged_bam_runs")
bam_files <- list.files(bam_dir, pattern=".bam$", full.names=TRUE)
sample_info <- data.frame(
  bam_files = bam_files <- list.files(bam_dir, pattern=".bam$", full.names=TRUE)) %>%
    dplyr::mutate(sample_name = str_replace(basename(bam_files),
                                            ".bam", ""),
                  treatment = str_replace(str_sub(sample_name, start=1L, end=-3L), "[^_]+_", "")) %>%
    dplyr::mutate(treatment = factor(treatment, levels=c("untreated", "DOX-pulse", "IFNg", "DOX-pulse_IFNg")))
```


## Esitmate the optimal read lengths
The code chunk below uses the `ribosomePrfilingQC` package to get the length of the RPFs and reveals optimal read lengths. Fig \@ref(fig:include-size-length) shows that the dominated RPF segment size ranges from 26 to 29 nt. 


```r
# (a) read length frequency
read_length_freq <- bplapply(sample_info$bam_files, function(x) {
  bam_file <- BamFile(x)
  p_site <- estimatePsite(bam_file, CDS, genome)
  pc <- getPsiteCoordinates(bam_file, bestpsite = p_site)
  read_length_freq <- summaryReadsLength(pc, widthRange = c(25:39), plot=FALSE)
})
names(read_length_freq) <- sample_info$sample_name

# (b) tidy data
length_freq <- map_dfr(names(read_length_freq), function(x) {
  as.data.frame(read_length_freq[[x]]) %>%
    dplyr::rename(length="Var1") %>%
    add_column(sample_name=x) %>%
    dplyr::mutate(order = as.numeric(length)) %>%
    dplyr::mutate(length = as.numeric(as.character(length)))
}) %>%
  left_join(dplyr::select(sample_info, sample_name, treatment),
            by="sample_name") %>%
  dplyr::arrange(treatment) %>%
  dplyr::mutate(sample_name = factor(sample_name, levels=unique(sample_name)))

# (c) plot
ggplot(length_freq, aes(x=length, y=Freq)) +
  geom_bar(stat="identity", width=0.7) +
  theme_bw() +
  facet_wrap( ~ sample_name, nrow=4) +
  labs(x="Read length", y="Frequency")
ggsave(file.path(fig_dir, "freqment_size_frequency.pdf"))
```


```r
knitr::include_graphics("images/freqment_size_frequency.pdf")
```

<div class="figure" style="text-align: center">
<embed src="images/freqment_size_frequency.pdf" title="Distribution of size of RPF segments" width="400px" height="400px" type="application/pdf" />
<p class="caption">(\#fig:include-size-length)Distribution of size of RPF segments</p>
</div>


## Distance from 5' end of reads to the start codon
The distance from 5' end of reads to the start codon of CDS can help to determine the best position of p-sites. In Figure \@ref(fig:distant-to-start-codon), the 5' end of reads are enriched at 13 position upstream from the start codon. This means the best p-sites is located at 13th nucleotide of the RPF segment and lots of ribosome are docking at the translation start sites.

Note that the `ribosomeProfilingQC::readsEndPlot()` function meant to make such a plot; however, the function itself has a flaw that fails to reverse the mapping for the genes on the negative strand. I forked the function and collected the mistake. (Later I will fork the pakcage and make it available on github, meanwhile, I am using `fork_readsEndPlot` from `scripts/fork_readsEndPlot.R`).

Code chunk below is the tool to tidy the distance data.

```r
.tidy_dist_data <- function(dist_list) {
  dist <- map_dfr(names(dist_list), function(x) {
    as.data.frame(dist_list[[x]]) %>%
    dplyr::rename(counts = `dist_list[[x]]`) %>%
    tibble::rownames_to_column(var = "dist") %>%
    tibble::add_column(sample_name = x) %>%
    dplyr::mutate(dist = factor(dist, levels=dist)) 
  }) %>%
    dplyr::left_join(dplyr::select(sample_info, sample_name, treatment), by="sample_name") %>%
    dplyr::arrange(treatment) %>%
    dplyr::mutate(sample_name = factor(sample_name, levels=unique(sample_name)))
}
```


Restricting the read length to 26 to 29 nt, the code below estimates the pileup of 5' end of reads 30 position up/down-stream from the start codon:

```r
read_length <- c(26:29)
# (a) distance to start codon [-29, 30]
start_codon_30 <- bplapply(sample_info$bam_files, function(x) {
  bam_file <- BamFile(x)
  fork_readsEndPlot(bam_file, CDS, toStartCodon=TRUE, readLen=read_length, window=c(-29, 30))
  #ribosomeProfilingQC::readsEndPlot(bam_file, CDS_pos, toStartCodon=TRUE, readLen=read_length,
  #             window= c(-29, 30))
})
names(start_codon_30) <- sample_info$sample_name

# (b) tidy data and hist (bar)
.tidy_dist_data <- function(dist_list) {
  dist <- map_dfr(names(dist_list), function(x) {
    as.data.frame(dist_list[[x]]) %>%
    dplyr::rename(counts = `dist_list[[x]]`) %>%
    tibble::rownames_to_column(var = "dist") %>%
    tibble::add_column(sample_name = x) %>%
    dplyr::mutate(dist = factor(dist, levels=dist)) 
  }) %>%
    dplyr::left_join(dplyr::select(sample_info, sample_name, treatment), by="sample_name") %>%
    dplyr::arrange(treatment) %>%
    dplyr::mutate(sample_name = factor(sample_name, levels=unique(sample_name)))
}

dist <- .tidy_dist_data(start_codon_30)
ggplot(dist, aes(x=dist, y=counts)) +
  geom_bar(stat="identity", width=0.7) +
  theme_bw() +
  labs(x="Distance from 5' end of reads to start codon", y="counts") +
  facet_wrap( ~ sample_name, nrow=4, scale="free") +
  geom_vline(xintercept = which(dist$dist == 1),
             linetype="dashed", alpha=0.3, show.legend=FALSE) +
  theme(axis.text.x=element_text(angle = 90, hjust = 1, vjust=0.5, size=4),
        panel.grid.major = element_blank(), #panel.grid.minor = element_blank(),
        panel.background = element_blank())
ggsave(file.path(fig_dir, "distance_from_5end_to_start_codon_30-fork-readsEndPlot.pdf"))#, width=8, height=6)
```


```r
knitr::include_graphics("images/distance_from_5end_to_start_codon_30-fork-readsEndPlot.pdf")
```

<div class="figure" style="text-align: center">
<embed src="images/distance_from_5end_to_start_codon_30-fork-readsEndPlot.pdf" title="Distance from 5 prime end reads to start codon reveals the best position of p-site: 13 nucleotide shift" width="400px" height="400px" type="application/pdf" />
<p class="caption">(\#fig:distant-to-start-codon)Distance from 5 prime end reads to start codon reveals the best position of p-site: 13 nucleotide shift</p>
</div>

## P-sites and reading frames 
We use `ribosomeProfilingQC::getPsiteCoordinates()` and `ribosomeProfilingQC::assigneReadingFrame()` to get p-sites coordinates and assign the reading frames within annotated CDS. We plotted the p-sites around the translation start sites and colored them by reading frames. Figure \@ref(fig:reading-frame-tss) ensures that the p-sites were correct and in-frame with the start codon. 


```r
reading_frame <- bplapply(sample_info$bam_files, function(x) {
  bam_file <- BamFile(x)
  p_site <- estimatePsite(bam_file, CDS, genome) # 13
  pc <- getPsiteCoordinates(bam_file, bestpsite = p_site)
  pc_sub <- pc[pc$qwidth %in% read_length]
  pc_sub <- assignReadingFrame(pc_sub, CDS)
  distance <- plotDistance2Codon(pc_sub)
})

names(reading_frame) <- sample_info$sample_name

# tidy reading_frame tool
.tidy_reading_frame <- function(rf_list) {
  rf <- map_dfr(names(rf_list), function(x) {
    rf_list[[x]] %>% as.data.frame(stringsAsFactors=FALSE) %>%
      dplyr::rename(index="Var1", Frequency="Freq") %>%
      dplyr::mutate(index=as.numeric(index), Frequency=as.numeric(Frequency)) %>%
      dplyr::mutate(frame = as.factor(index %% 3)) %>%
      add_column(sample_name = x)
  })
}  

# ggplot
tidy_rf <- .tidy_reading_frame(reading_frame) %>%
  left_join(dplyr::select(sample_info, sample_name, treatment), by="sample_name") %>%
  dplyr::arrange(treatment) %>%
  dplyr::mutate(sample_name = factor(sample_name, levels=unique(sample_name)))

ggplot(tidy_rf, aes(x=index, y=Frequency, fill=frame)) +
    geom_bar(stat="identity", width=0.7) +
    theme_minimal() +
    facet_wrap( ~ sample_name, nrow=4, scale="free") +
    theme(legend.position=c(0.25, 0.93), legend.key.size = unit(0.3, 'cm'), 
          axis.text.x=element_text(size=5), panel.grid.major = element_blank()) +
    labs(x="P site relative to start codon", y="counts") +
    scale_x_continuous(breaks=seq(0, 50, 3)) +
    scale_fill_brewer(palette="Dark2")
ggsave(file.path(fig_dir, "reading_frame_psite_to_start_codon.pdf"), width=8, height=6)   
```

```r
knitr::include_graphics("images/reading_frame_psite_to_start_codon.pdf")
```

<div class="figure" style="text-align: center">
<embed src="images/reading_frame_psite_to_start_codon.pdf" title="Reading frames of p-sites on annotated CDS and around start codon" width="400px" height="400px" type="application/pdf" />
<p class="caption">(\#fig:reading-frame-tss)Reading frames of p-sites on annotated CDS and around start codon</p>
</div>

## Trinucleotide periodicity on transcripts
Here, we constructed a reading frame frequency plot and meta-gene plot of p-sites coverage, colored by reading frames, on annotated 5' UTR, CDS, and 3' UTR regions. Figure \@ref(fig:reading-frame-freq-transcripts) and Figure \@ref(fig:meta-gene-coverage) confirmed that (1) the enriched RPFs docked on the translation start sties, and (2) the trinucleotide footprint periodicity only occurred on the CDS. 

Because the `ribosomeProfilingQC::assigneReadingFrame()` function only assigns reading frames for p-sites on the annotated CDS, I wrote [tools](https://https://github.com/FredHutch/DUX4-IFNg-ribosome-footprints/scripts/tools_assign_utr_frame.R) to assign reading frame to p-sites that are exclusively lay on the annotated UTR regions. Giving an overview of the trinucleotide periodicity on the whole transcripts this [script](https://https://github.com/FredHutch/DUX4-IFNg-ribosome-footprints/scripts/10C-manuscript-figures-periodicity-reading-frames.R) constructed a meta-gene p-sites coverage in 5' UTR, CDS, and 3' UTR regions, as shown in Figure \@ref(fig:meta-gene-coverage). 


```r
knitr::include_graphics("images/frame_frequency_by_regions_average_over_treatment.pdf")
```

<div class="figure" style="text-align: center">
<embed src="images/frame_frequency_by_regions_average_over_treatment.pdf" title="Reading frame frequency on 5 prime UTR, CDS, and 3 prime UTR" width="400px" height="170px" type="application/pdf" />
<p class="caption">(\#fig:reading-frame-freq-transcripts)Reading frame frequency on 5 prime UTR, CDS, and 3 prime UTR</p>
</div>

```r
knitr::include_graphics("images/reading_frame_periodicity_by_treatment_norm.pdf")
```

<div class="figure" style="text-align: center">
<embed src="images/reading_frame_periodicity_by_treatment_norm.pdf" title="Meta-gene p-sites coverage" width="400px" height="200px" type="application/pdf" />
<p class="caption">(\#fig:meta-gene-coverage)Meta-gene p-sites coverage</p>
</div>
