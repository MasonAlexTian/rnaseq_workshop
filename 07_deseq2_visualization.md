# Differential Expression

In this tutorial, you will:

* Make use of the raw counts you generated in the previous session using `htseq-count`.
* Use DESeq2: a Bioconductor package designed specifically for differential expression of count-based RNA-seq data.
* Visualize differentially expressed genes (features).

## Main objectives of differential gene expression analysis

RNA-seq is often used to compare one tissue type to another (liver vs. spleen), or experimental group vs. control group sample. There are many specific genes transcribed in the liver but not in the spleen or vice versa. This is called "a difference in library composition".

Open the following R script with `rstudio` that performs an RNA-Seq differential analysis.  

```bash
rc
rstudio runDESeq2FromFeatureCount2.r &
```

First, the R library called `apeglm` is required. Let us first install it in R:

```r
source("http://bioconductor.org/biocLite.R")
biocLite("apeglm")    # only if devtools not yet installed
library(apeglm) #load it & check if the module was installed successfully
```  

### Preparing DESeq2 input
1. Read the **raw** count matrix we generated in the previous session.
1. Assign which sample belongs to which group.
1. Retain only genes where half of the samples have at least 10 reads.

### DESeq2
DESeq2 (or edgeR) handles both library size and composition issues using log normalization and negative binomial distribution model. Refer to [DESeq2 manual](http://bioconductor.org/packages/devel/bioc/vignettes/DESeq2/inst/doc/DESeq2.html#how-do-i-use-vst-or-rlog-data-for-differential-testing) for more details.

#### Scaling Factor (normalize library size)
Given a matrix with *J* columns (samples) and *n* rows (genes), we estimate the size factors as follows:

Each column element is divided by the geometric means of the rows. For each sample, the median (or, if requested, another location estimator) of these ratios (skipping the genes with a geometric mean of zero) is used as the size factor for this column.

Let *K<sub>g,j</sub>* be a gene count at gene *g* and sample *j*. The scaling factor for sample *j* is thus obtained as:

![s_j=median\left(\frac{K_{g,j}}{\left(\prod_{j=1}^{J}K_{g,j}\right)^{1/J}}\right)](http://latex.codecogs.com/png.latex?s_j&space;=&space;median\left\(\frac{K_{g,j}}{\left(\prod_{j=1}^{J}K_{g,j}\right&space;)^{1/J}}\right\))

This function is implemented in `DESeq2::estimateSizeFactors()`.

#### Q7.1: Compare row counts boxplot and density plot before/after applying the scaling factor

#### Modeling read counts

When working with biological replicates, more variations are intrinsically expected. Indeed, the measured expression values for each gene are expected to fluctuate more importantly, due to the combination of biological and technical factors:
* Inter-individual variations in gene regulation
* Sample purity
* Cell-synchronization issues
* Responses to an environment (e.g. heat-shock).

The Poisson distribution has only one parameter indicating its expected mean: *λ*. The variance of the distribution equals its mean *λ*. In most cases, the Poisson distribution is not expected to fit very well with the count distribution in biological replicates since we expect some over-dispersion (greater variability) due to biological noise.

As a consequence, when working with RNA-Seq data, many of the current approaches for differential expression calling rely on the negative binomial distribution (note that this holds true also for other \*-Seq approaches, e.g. ChIP-Seq with replicates).

#### What is the negative binomial distribution?

The negative binomial distribution is a discrete distribution that gives us the probability of observing *x* failures before a target number of successes *n* is obtained. As we will see later, the negative binomial distribution can also be used to model over-dispersed data (in this case, this overdispersion is relative to the Poisson model).

The probability of *x* failures before *n* success given a Bernoulli trial with a probability *p* of success is given by:

![P_{negbin}(x;n,p) = \binom{x+n-1}{x}p^n(1-p)^x](https://latex.codecogs.com/gif.latex?P_{NegBin}(x;n,p)&space;=&space;\binom{x&plus;n-1}{x}p^n(1-p)^x)

Use `dbbinom()` to show the probability density of *x* failure before *n=2* success with *p=1/6*.

```r
# introduce a concept of negative binomial dist
n <- 2
p <- 1/6
x <- 0:30
plot(x, dnbinom(x, size=n, prob=p), type="h", col="blue", lwd=2,
     main="Negative binomial density",
     ylab="P(x; n,p)",
     xlab=paste("x = number of failures before", n, "successes"))

# Expected value
q <- 1-p
ev <- n*q/p

abline(v=ev, col="darkgreen", lwd=2)

# Variance
v <- n*q/p^2

arrows(x0=ev-sqrt(v), y0 = 0.04, x1=ev+sqrt(v), y1=0.04, col="brown",lwd=2, code=3, length=0.2, angle=20)
```

#### Modelling read counts through a negative binomial
To perform differential expression calls, DESeq will assume that, for each gene, the read counts are generated by a negative binomial distribution. One problem here will be to estimate, for each gene, the two parameters of the negative binomial distribution: mean and dispersion. The parameter estimation is implemented in two steps: `estimateDispersions()` and negative binomial Generalized Linear Model (GLM) fitting. Refer to [Moderated estimation of fold change and dispersion for RNA-seq data with DESeq2](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-014-0550-8) for the theory behind the DESeq2.

```r
## Performing estimation of dispersion parameter
dds.disp <- estimateDispersions(dds.norm)

## A diagnostic plot which
## shows the mean of normalized counts (x axis)
## and dispersion estimate for each genes
plotDispEsts(dds.disp)
```

When the number of available samples or the count itself is insufficient to obtain a reliable estimator of the variance for each gene, DESeq will apply a shrinkage strategy (e.g., log fold change estimate), which assumes that *"counts produced by genes with similar expression level (counts) have similar variance"* (note that this is a strong assumption). The method is implemented in `lfcShrink(type="apeglm")`. Refer to "[Heavy-tailed prior distributions for sequence count data: removing the noise and preserving large differences](https://www.biorxiv.org/content/early/2018/04/17/303255)" for more detail. The result is useful for visualization and ranking of genes.

#### Performing differential expression call
Now that a negative binomial model has been fitted for each gene, the `nbinomWaldTest()` function can be used to test for differential expression. The output is a data.frame which contains nominal p-values, as well as FDR values (corrections for multiple tests are computed with the Benjamini–Hochberg procedure).

```r
alpha <- 0.05
wald.test <- nbinomWaldTest(dds.disp)
res.DESeq2 <- results(wald.test, alpha=alpha, pAdjustMethod="BH")

## What is the object returned by nbinomTest()
class(res.DESeq2)
res.DESeq2 <- res.DESeq2[order(res.DESeq2$padj),]

hist(res.DESeq2$padj, breaks=20, col="grey", main="DESeq2 p-value distribution", xlab="DESeq2 P-value", ylab="Number of genes")

```

### All-in-one with DESeq()
* Estimate size factors and dispersions.
* Fit negative binomial distribution model.
* Perform wald test
  * Note that the function uses the estimated standard error of a log2 fold change (lfcSE) to test if it is equal to zero.

Open DESeq2 report file or we will analyze this via heatmap below.

### Hierarchical Clustering
1. Computes the distance between each pair of samples.
1. Performs hierarchical clustering.

### PCA Plot
Principal Component Analysis (PCA) is a method of dimensionality reduction/feature extraction that transforms the data from a d-dimensional space into a new coordinate system of dimensions *p*, where *p<=d*.
Here, we use only two PCs.

### Heatmap
1. Load the DESeq2 output file.
1. Focus on significant differentially-expressed genes.
1. Load the log2(TPM2) file.
1. Generate a heatmap from the log2(TPM2) counts only reported in the refined DESeq2 output table.

Let us run the R script to generate all tables and figures for DE analysis:
```bash
Rscript ./runDESeq2FromFeatureCount2.r
```

Open the image files in the output directory `expr_output`:
```bash
rc
cd expr_output
magick display <image_file> &
```

### Task 1
We prepared another read count matrix file for you.
1. Extract this tar file into your project directory:

  `/mnt/isilon/data/w_QHS/hwangt-share/Datasets/Informatics_workshop/rnaseq/test_data_for_DESeq2.tar.gz`
1. Summarize a read count matrix on the features.
1. Repeat the same analysis done above.
1. Compare your results (table and figures) with the ones in your projects directory:

  `~/projects/test_data_for_DESeq2/DESeq2_results`

### Task 2
Use the DESeq2 report to perform pathway analysis so that you can see which pathway turns on or off in the experimental samples by comparing to the control samples.

- https://stephenturner.github.io/deseq-to-fgsea/

### Resources
- https://cran.r-project.org/doc/contrib/Torfs+Brauer-Short-R-Intro.pdf
- http://bioconductor.org/packages/devel/bioc/vignettes/DESeq2/inst/doc/DESeq2.html#differential-expression-analysis
- http://pedagogix-tagc.univ-mrs.fr/courses/ASG1/practicals/rnaseq_diff_Snf2/rnaseq_diff_Snf2.html

### Up next
[Kallisto](08_kallisto.md)
