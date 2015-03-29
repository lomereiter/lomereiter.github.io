---
layout: post
title: '[LMM] literature overview: performance'
---

Now let's focus on performance optimizations on the computing side. The methods discussed below still use exact formulas, but organize computations in a smart way. They happen to be implemented in `omicABEL` package (GPLv3) and its fork, `cuOmicABEL`.

## omicABEL

I start with [High-Performance Mixed Models Based Genome-Wide Association Analysis with omicABEL software](http://www.ncbi.nlm.nih.gov/pmc/articles/PMC4329600/pdf/f1000research-3-5195.pdf) (August 2014), where two methods are compared, one using eigendecomposition and the other Cholesky decomposition.

To speed up the operations, tested SNPs are packed into batches, so that instead of relatively slow matrix-vector operations (BLAS-2) matrix-matrix are employed (BLAS-3). This suggestion first appeared in the paper [Solving Sequences of Generalized Least-Squares
Problems on Multi-threaded Architectures](http://www.sciencedirect.com/science/article/pii/S0096300314002951) (2014), also there's a less polished version of it on [arxiv.org](http://arxiv.org/pdf/1210.7325.pdf) (2012).

It turns out that the eigendecomposition-based method is faster in case of multiple trait analysis, but Cholesky beats it when a single trait is analysed.

The authors test the method against FaST-LMM, and announce the victory:

> In case of single-trait analysis, our results are somewhat less
> impressive, and our CLAK-Chol solution outperforms advanced
> current methods (e.g. FaST-LMM) by about one order of magnitude 
> for large-sample-size problems. It is worth mentioning that the
> latter speed-up becomes possible because we show that for 
> single-trait GWAS problems our CLAK-Chol algorithm is superior to
> CLAK-Eig, but other current methods are actually implementing
> solutions similar to our CLAK-Eig algorithm to address the 
> single-trait GWAS problem.

They also provide an excellent tutorial [here](http://www.genabel.org/sites/default/files/tutorials/OmicABEL/exampleOfUse.html) that I followed along. It's slightly outdated, though:

* The program name was changed from `CLAK-GWAS` to `OmicABEL`, and the links to binaries are found [here](http://www.genabel.org/packages/OmicABEL)
* The name of output file produced by `reshuffle` tool has been *reshuffled* into `data_chi.txt` from `chi_data.txt`, and the option has been renamed from `--chi2` to `--chi` and now requires a (positive integer) threshold, so I used `--chi2=1`. I'm a bit annoyed that I can't specify a float or zero, but OK, nobody cares about non-significant SNPs.

The results are indeed well correlated, even for large values of $\chi^2$, and FaST-LMM is indeed beaten (timings on my laptop are similar to those in the tutorial).

### Extremely large datasets

There is another related paper (almost the same set of authors) focusing more on the algorithmic & computational aspects: <br/>[High performance solutions for big-data GWAS](http://www.sciencedirect.com/science/article/pii/S0167819114001161); on [arxiv.org](http://arxiv.org/pdf/1403.6426v1.pdf) since March 2014.

It goes without saying that there is no need to load all SNPs at once, and out-of-core algorithm is provided, featuring asynchronous I/O so that I/O and computations overlap.

But the authors go further and consider how to process huge number of individuals. All algorithms require storing $N\times N$ kinship matrix (or its eigenvectors, or Cholesky factors), which might not fit into memory of a single machine. The authors suggest use of [Elemental](http://libelemental.org/) library and provide a description of how to distribute matrix blocks across nodes.

## cuOmicABEL

Finally, there actually IS a tool allowing to speed up GWAS with GPU!
In 2012, Lucas Beyer from RWTH Aachen University defended his thesis, [Exploiting Graphics Accelerators for Computational Biology](http://lucasb.eyer.be/academic/gwas/lbeyer-thesis-gwas.pdf). (It should not be surprising that the papers discussed above also include people from the same university.) Later, a paper appeared on `arxiv.org`, [Streaming Data from HDD to GPUs for Sustained Peak Performance](http://arxiv.org/pdf/1302.4332v1.pdf). As the name suggests, a clever buffering scheme is proposed, and the testing shows that it leads to near-ideal scaling. 

The work is available on Github in the form of fork of `omicABEL`: [https://github.com/lucasb-eyer/cuOmicABEL](https://github.com/lucasb-eyer/cuOmicABEL); it doesn't seem to be merged, and there's been no progress made since spring 2013. Hopefully it works.