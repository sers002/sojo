What is SOJO?
-------------

SOJO stands for **S**election **O**perator for **JO**intly analyzing
multiple variants. It is a tool for implementing Least Absolute
Shrinkage and Selection Operator (LASSO) using GWAS summary statistics.
LASSO was introduced and applied to variable selection problems in
various disciplines because of its better interpretability and
prediction accuracy. More specifically, Instead of only considering the
square loss function
![](http://mathurl.com/yctode9q.png), LASSO
takes the ℓ<sub>1</sub>-norm regularization ![](http://mathurl.com/y9k4jx47.png)
into account and solves

<p align="center"> 
<img src="http://mathurl.com/ycl9zc3k.png">
</p>

where the tuning parameter *λ* ≥ 0. The regularization term makes LASSO
allow large coefficients only when they lead to substantially better
fit. Here, we implement SOJO to detect multiple associations in LD at
the same loci based on LASSO result. The LASSO result at any tuning
parameter can be approximated by using (i) the covariance structure
between variants and the trait and (ii) the LD structure between
variants. The method is described in:

[Ning et al. A Selection Operator for Summary Association Statistics Reveals Allelic Heterogeneity of Complex Traits. The American Journal of Human Genetics, 2017](http://www.cell.com/ajhg/fulltext/S0002-9297(17)30419-6)

What data are required?
-----------------------

* GWAS summary statistics of SNPs at a locus for a
trait. The data frame of GWAS summary statistics should contain columns
for variant names (column name `SNP`), effect alleles (column name
`A1`), reference alleles (column name `A2`), allele frequencies of A1
(column name `Freq1`), effect sizes (column name `b`), standard errors
(column name `se`), and sample sizes (column name `N`). An example data
frame is given later.

* A reference LD correlation matrix including SNPs at the locus and its corresponding reference alleles. Users can download reference LD correlation matrices and the reference alleles used to compute the LD matrices from https://www.dropbox.com/home/sojo%20reference%20ld%20matrix. 
These LD matrices are based on 612,513 chip markers in Swedish Twin Registry. The function will then take overlapping SNPs between summary statistics and reference LD matrix. If chip markers are insufficient for your study, in this manual, we also provide commands to compute LD matrix and reference allele information based on 1000 Genomes European-ancestry samples.

Installation
------------

Run the following command in R to install the **sojo** package:

    install.packages("sojo", repos = "http://R-Forge.R-project.org")

For Mac and Linux users, please download the source .tar.gz from https://r-forge.r-project.org/R/?group_id=2030
and install via:

    R CMD INSTALL sojo_1.0.tar.gz

in your terminal.

In R, load the package via:

    library(sojo)

or

    require(sojo)

LASSO using Summary Statistics
------------------------------

#### Example Summary Statistics

You need to load a data frame of GWAS summary statistics for a trait
into your working directory. Let us demonstrate this via an example
included in the package. Here, we have the summary statistics for height
across 963 variants around rs11090631 on chromosome 22. The top of the
summary statistics file looks like:

    data(sum.stat.raw)
    head(sum.stat.raw)

    ##          SNP A1 A2 Freq1       b     se      N
    ## 1  rs1022622  C  G 0.942  0.0072 0.0050 241337
    ## 2  rs2024708  A  G 0.833  0.0052 0.0038 253216
    ## 3  rs2073239  A  G 0.833  0.0051 0.0038 253248
    ## 4  rs5765102  A  G 0.167 -0.0061 0.0037 252230
    ## 5  rs5766231  A  G 0.833  0.0052 0.0038 253216
    ## 6 rs11703912  A  G 0.833  0.0052 0.0038 252265

#### Download the reference LD correlation matrix

Now we need the reference LD information at the locus where rs11090631 is located in. For European-ancestry data, we provide two sources of LD information so that you do not need to compute your own LD matrix.

1. The Swedish Twin Registry 

We can download the LD matrix and reference allele imformation for chromosome 22 directly by:

    download.file("https://www.dropbox.com/s/ty1udfhx5ohauh8/LD_chr22.rda?raw=1", 
    destfile = paste0(find.package('sojo'), "/LD_chr22.rda"))
    
Then load it into environment:

    load(file = paste0(find.package('sojo'), "/LD_chr22.rda"))
    
2. The 1000 Genomes Project

We did not upload the LD matrix from the 1000 Genomes Project because the number of variants is much larger here. However, we can use the following commands to get the LD matrix and reference allele imformation for chromosome 22. **Note: [plink 1.9](https://www.cog-genomics.org/plink2) is needed for the following commands.**

Firstly, we can download and unzip genotypes of 1000 Genomes European-ancestry samples (provided by LDSC project) by

    download.file("https://data.broadinstitute.org/alkesgroup/LDSCORE/1000G_Phase3_plinkfiles.tgz", 
    destfile = paste0(find.package('sojo'), "/1000G_Phase3_plinkfiles.tgz"))
    untar(paste0(find.package('sojo'), "/1000G_Phase3_plinkfiles.tgz"),exdir=find.package('sojo'))
    
Then we specify the path to plink (**Note: Please change the path below to your own path to the plink executable file**) by 

    path.plink <- "path/to/plink/executable/file/plink"
    
and the path to 1000 Genomes data by

    path.1kG <- paste0(find.package('sojo'),"/1000G_EUR_Phase3_plink")
     
Now, we can get the LD matrix and reference allele imformation for the SNPs in this sumamry statistics data frame by

    snps <- sum.stat.raw$SNP
    write.table(snps, file = paste0(snps[1],"_snp_list.txt"), quote = F, row.names = F, col.names = F)
    chr <- 22

    system(paste0(path.plink," -bfile ", path.1kG,"/1000G.EUR.QC.",chr," --r square --extract ", snps[1], "_snp_list.txt --out ", snps[1], " --noweb"))
    system(paste0(path.plink," -bfile ", path.1kG,"/1000G.EUR.QC.",chr," --freq --extract ", snps[1], "_snp_list.txt --out ", snps[1], " --noweb"))

    LD_1kG <- as.matrix(read.table(paste0(snps[1], ".ld")))
    maf_1kG <- read.table(paste0(snps[1], ".frq"), header = T)
    snp_ref_1kG <- maf_1kG[,"A2"]
    names(snp_ref_1kG) <- maf_1kG[,"SNP"]
    colnames(LD_1kG) <- rownames(LD_1kG) <- maf_1kG$SNP


#### A Simple sojo analysis

Once the data are successfully loaded and all necessary columns are
present, the LASSO solution can be computed by:

    res <- sojo(sum.stat.raw, LD_ref = LD_mat, snp_ref = snp_ref, nvar = 20)

The result is a list with two sub-objects `$lambda.v` and `$beta.mat`.
By setting `nvar = 20`, the computation stops when the model include 20
variants with non-zero coefficients. The vector of lambdas when a
variant is added into or removed from the model can been seen by:

    res$lambda.v

    ##  [1] 0.012017 0.011131 0.009834 0.009622 0.009194 0.007176 0.006401
    ##  [8] 0.005839 0.005157 0.004852 0.004730 0.004518 0.004410 0.004284
    ## [15] 0.004102 0.004016 0.003913 0.003896 0.003830 0.003814 0.003618
    ## [22] 0.003569 0.003531 0.003522 0.003510

We can check the LASSO solutions for some variants at lambdas among
`lambda.v`:

    snp_selected <- which(res$beta.mat[,5] != 0)
    res$beta.mat[snp_selected,1:4]

    ## 5 x 4 sparse Matrix of class "dgCMatrix"
    ##           beta      beta       beta      beta
    ## rs2142661    . .         -6.986e-18 3.330e-04
    ## rs9614670    . 2.596e-17  1.408e-03 1.379e-03
    ## rs6006753    . 1.894e-03  3.302e-03 3.589e-03
    ## rs714022     . .          .         1.920e-18
    ## rs8141212    . .          .         .

The LASSO path plot can be obtained by:

    matplot(log(res$lambda.v), t(as.matrix(res$beta.mat)), lty = 1, type = "l", 
        xlab = expression(paste(log, " ", lambda)), ylab = "Coefficients", main = "Summary-level LASSO")

<img src="Images/example.png" width="700">
LASSO solution at some specific tuning parameters can also be computed
via:

    res2 <- sojo(sum.stat.raw = sum.stat.raw, LD_ref = LD_mat, snp_ref = snp_ref, lambda.vec = c(0.004,0.002))

For Help
--------

For direct R documentation of `sojo` function, you can simply use
question mark in R:

    ?sojo

If you have specific questions, you may email the maintainer of sojo via
<zheng.ning@ki.se>.

Citation
--------

If you use the R package `sojo`, please cite

[Ning et al. A Selection Operator for Summary Association Statistics Reveals Allelic Heterogeneity of Complex Traits. The American Journal of Human Genetics, 2017](http://www.cell.com/ajhg/fulltext/S0002-9297(17)30419-6)