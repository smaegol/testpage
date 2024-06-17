Copied from https://github.com/LRB-IIMCB/yeast_deadenylation_methods_chapter/blob/main/README.md as a template:

# Protocol

Having all reads with called poly(A) lengths, it is possible to continue
with statistical data analysis. As this is direct RNA sequencing, each
read represents a single transcript present in the library. At this
point there are multiple routes of analysis possible, depending on the
experimental setup chosen at the beginning. Here we will cover only the
basic analysis possible with obtained data.

## Load required libraries

This will load all software packages required for processing of data
included in this dataset

<details open>
<summary>Code</summary>

``` r
# load libraries
library(tidyverse)
library(nanotail)
library(ggplot2)
```

</details>

## Load metadata

Load the example metadata tables for both *S. cerevisiae* and *S. pombe*
spike-in data into R session using:

<details open>
<summary>Code</summary>

``` r
metadata <- read.table("metadata_heat.csv",sep=",",header=T)
metadata_pombe <- read.table("metadata_heat_pombe.csv",sep=",",header=T)
```

</details>

<div>

> **Note**
>
> Metadata for multiple samples can be stored in data.frame object, with
> the required columns:
>
> - polya_path (containing the path to nanopolish output file)
> - sample_name.
>
> Additional columns may contain supplementary information describing
> experimental conditions and will be included in the analysis table
> loaded in the next step of the protocol.

</div>

## Load poly(A) lengths data

Data are loaded into single data.frame using read_polya_multiple()
function from the nanotail package, taking metadata table as an
argument.

<details open>
<summary>Code</summary>

``` r
# load data
polya_data_table <- read_polya_multiple(metadata) 

polya_data_table$group <- factor(polya_data_table$group,levels=c("t0","t2","t6","t10","t18"))
polya_data_table_pombe <- read_polya_multiple(metadata_pombe) 
```

</details>

<div>

> **Note**
>
> There is also read_polya_single() function, which loads data from the
> single experiment to the environment.

</div>

## Basic QC

Basic QC can be done using functions included in the nanotail package.
This will summarize data based on the content of qc_tag column of
nanopolish output files.

<div>

> **Note**
>
> Qc_tag column of nanopolish polya output summarizes the reliability of
> poly(A) lengths calculation for each read. Possible categories
> include:
>
> - PASS – read with correct segmentation, reliable calculation of
>   poly(A) length,
> - ADAPTER – the read was stuck for too long at the adapter part, so
>   the read rate estimation, and thus poly(A) length calculation may be
>   unreliable,
> - SUFFCLIP – 3’ terminal part of a read (before poly(A) tail) was not
>   mapped to a reference, so the assignment to reference may be
>   inaccurate,
> - NOREGION – no poly(A) region was found in the raw signal,
> - READ_FAILED_LOAD – there was an issue with reading raw data from
>   fast5 file.

</div>

<details open>
<summary>Code</summary>

``` r
nanopolish_qc <- get_nanopolish_processing_info(polya_data=polya_data_table,grouping_factor = "sample_name") 

#polya_data is the table loaded in point 4. 

# grouping_factor can be any of the metadata columns in the metadata table. Setting to NA will produce a summary of the whole dataset 

plot_nanopolish_qc(nanopolish_processing_info=nanopolish_qc,frequency=F) 
```

</details>

![](README_files/figure-commonmark/nanopolish-qc-1.png)

As we can see, a large proportion o reads are assigned to the SUFFCLIP
category. This is due to the reference lacking UTR sequences. As this
should not bias poly(A) length estimations, reads with such a category
can be safely included in the further processed dataset. However,
caution should be taken every time such QC plot shows a large proportion
of categories other than PASS.

<div>

> **Warning**
>
> If there is a large proportion (\>5%) of READ_FAILED_LOAD this means
> there were issues with raw data accessibility. After checking that all
> raw fast5 files are accessible nanopolish analysis should be re-run.

</div>

<div>

> **Warning**
>
> There should not be many NOREGION reads as the library preparation
> protocol selects poly(A)-containing RNAs. Such a category usually
> means that sequencing read was incorrectly processed by the sequencing
> software (MinKNOW) and should be discarded from the analysis.

</div>

<div>

> **Warning**
>
> A large proportion of ADAPTER reads means that there are problems with
> the sequencing itself and, even if such reads could be considered for
> further analysis it is advisable to rerun the sequencing of such
> samples.

</div>

## Data filtering

In the next step, reads which failed internal nanopolish QC (assigned to
categories NOREGION, READ_FAILED_PASS, or ADAPTER) can be removed. It
can be done using dplyr package:

<details open>
<summary>Code</summary>

``` r
polya_data_filtered <- polya_data_table %>% dplyr::filter(qc_tag %in% c("PASS","SUFFCLIP")) 
```

</details>

<div>

> **Note**
>
> SUFFCLIP reads were included as we expect them based on the reference
> transcriptome. However, in most cases only PASS reads should be
> included in the filtered dataset.

</div>

## poly(A) lengths distribution

Next, we can show global distribution of poly(A) tail lengths on a
density plot using plot_polya_distribution() function:

<details open>
<summary>Code</summary>

``` r
plot_polya_distribution(polya_data=polya_data_filtered, groupingFactor="group",show_center_value="median")
```

</details>

![](README_files/figure-commonmark/plot-polya-distribution-1.png)

<div>

> **Note**
>
> If spike-ins were added to the library, their poly(A) distributions
> should also be examined to identify any technical issues that can bias
> the DRS output. Samples for which *S. pombe* (or other spike-in)
> poly(A)-tail lengths strongly diverge should be discarded.

</div>

For multiple samples, it is better to use a violin plot instead”

<details open>
<summary>Code</summary>

``` r
plot_polya_violin(polya_data = polya_data_filtered,groupingFactor = "group",add_boxplot=T,transcript_id=NULL)  
```

</details>

![](README_files/figure-commonmark/plot-polya-violin-1.png)

## Statistics

Next, find transcripts with significant change in poly(A) lengths
between conditions using kruskal_polya() function (from the nanotail
package). It uses Kruskall-Wallis test for comparison between multiple
groups.

<div>

> **Note**
>
> Computation of statistics can take several minutes to finish

</div>

<details open>
<summary>Code</summary>

``` r
polya_data_stats<-kruskal_polya(input_data=polya_data_filtered,grouping_factor="group",transcript_id="transcript") 
```

</details>

<div>

> **Tip**
>
> The result table can be reviewed with the command:
>
>     View(polya_data_stats %>% dplyr::filter(padj<0.05)) 

</div>

Number of transcripts showing a significant change in poly(A) lengths
can be assessed with:

<details open>
<summary>Code</summary>

``` r
polya_data_stats %>% dplyr::filter(padj<0.05) %>% count() 
```

</details>

|   n |
|----:|
| 150 |

## Single transcripts

Poly(A) lengths for individual transcripts can be viewed using
plot_polya_distribution() or plot_polya_violin() functions, similar to
global distribution.

The transcript identifier can be specified with “transcript_id” argument
and may contain multiple transcript ids in form of vector.

Example for YMR251W-A_mRNA transcript, which is the most significant hit
from statistical analysis performed in a previous step:

<details open>
<summary>Code</summary>

``` r
plot_polya_violin(polya_data = polya_data_filtered,groupingFactor = "group",transcript_id="YMR251W-A_mRNA")  
```

</details>

![](README_files/figure-commonmark/plot-single-transcript-violin-1.png)

## Per-transcript summary

Next, calculate the per-transcript summary of poly(A) length
distributions. This will produce mean, median, and quantile
(0.03,0.05,0.95,0.97) values for each transcript at each condition. Such
numeric values can nicely show changes in poly(A) length between
analyzed conditions as they represent poly(A) length dynamics with
better sensitivity than just looking at mean or median values.

<details open>
<summary>Code</summary>

``` r
polya_summary <- summarize_polya_per_transcript(polya_data = polya_data_filtered,groupBy="group",transcript_id="transcript",quantiles=c(0.03,0.05,0.95,0.97),summary_functions=c("mean","median"))
```

</details>

<div>

> **Note**
>
> Most yeast cellular poly(A)-tail profiles are normal with the mean and
> median located between 20 and 40 adenosines. Since from other studies
> we know that the newly made poly(A)-tail length is around 50-60
> adenosines with the upper adenylation limit of 200<sup>1,5</sup>, the
> mean and median values most likely represent RNAs deadenylated in the
> cytoplasm.
>
> Distribution of yeast poly(A)-tail quantiles values will range between
> 50-60 for the upper quantiles (e.g. 75-99th), and as follows those
> values will represent newly-made mRNAs, which might be nuclear or
> freshly exported to the cytoplasm. The 50th and neighboring quantiles
> will be similar to the mean and median, while the lower quantiles
> 5-20th will represent old and strongly deadenylated mRNAs, for which
> the poly(A)-tail lengths will be close to the DRS detection limit
> (around 5-10 adenosines).
>
> We routinely examine quantiles: 0.05, 0.10, 0.15, 0.50, 0.75, 0.80,
> 0.85, 0.90, and 0.95.

</div>

Obtained values can be plotted to show changes in selected transcripts.
Quantiles distribution can be plotted with function plot_quantiles()

<details open>
<summary>Code</summary>

``` r
plot_quantiles(summarized_data=polya_summary,transcript_id= "YMR251W-A_mRNA",groupBy="group") 
```

</details>