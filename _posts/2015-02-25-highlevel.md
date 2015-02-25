---
layout: post
title: '[FaST-LMM] high-level overview of the codebase, part 1'
---

Following the meet-in-the-middle approach, after thorough delving into the algorithmic core of the FaST-LMM,
I decided to finally run the code on test data and see what it does from the user point of view :)

Python notebook infrastructure, and the provided document FaST-LMM.ipynb help immensely with understanding how to run the analysis.
The notebook will serve as a guide through the available functionality.

## Required packages and modules
- python2
- cython
- pandas
- scipy
- scikit-learn
- statsmodels
- pysnptools

(All but the last one, I installed via Pacman)

## LMM(all)

The first approach is called 'traditional'. In order to exclude the tested SNP from the set on which null model is built, it simply skips the whole chromosome when building the model. It's less sophisticated than the following methods, but has less power (in the statistical sense).

Now let's go into depth and discover what's going under the hood in the straightforwardly named `single_snp_leave_out_one_chrom` function.

### single_snp_leave_out_one_chrom

The code is located in `fastlmm/association/single_snp.py`.
It's very simple to grasp, for each chromosome it builds a kinship matrix, and tests SNPs on this chromosome using the built matrix.
Internally it uses the LMM class from `lmm_cov.py` (which lacks documentation, or at least I couldn't find the paper with algorithm descriptions so far)

### Input formats

- SNPs are read with the aid of `pysnptools.snpreader.Bed`. The name is a bit deceiving, though. In fact, it reads three files with `.bed`, `.bim`, and `.fam` extensions. Also this BED format has nothing to do with UCSC BED; its name stands for 'Binary PED' used by PLINK ([http://pngu.mgh.harvard.edu/~purcell/plink/data.shtml](http://pngu.mgh.harvard.edu/~purcell/plink/data.shtml)). Non-binary PED and MAP formats described there are simple enough and don't deserve any more attention.

- Phenotype data is read via `pysnptools.util.pheno.loadPhen` function. It's in yet another PLINK format, mentioned on the same webpage, and contains three columns with family IDs, individual IDs, and phenotype values.

- As mentioned in the `.ipynb` document, "The covariate file also uses this format (with additional columns for multiple covariates)"

### Optional 'covar' argument

If it's provided, it's used as the **X** matrix (without the last column of ones). If not, **X** is just a vector of ones.

### Final notes
- The algorithm seems intended to be used with $G_0$ only, so that $K = G_0G_0^T$. Although the function has the optional $G_1$ parameter, comments indicate that this extra functionality hasn't been tested.

## LMM(all + select)

The next algorithm is more complicated. The principal idea is to improve statistical power by removing from GSM those SNPs that are uncorrelated with phenotypes.

A little bit more formal and math-inclined summary (taken from one of the papers) is as follows:

> The two main steps of FaST-LMM-Select are ranking SNPs by lin. reg.
> P-values to form the GRM with the top-ranked SNPs and then calculating association statistics in a mixed-model framework, using this
> GRM

Mixed-model basically means that the matrix $K$ is now a mix of $G_0G_0^T$ and $G_1G_1^T$ where $G_1$ corresponds to the selected SNPs.


- The first step is feature selection

{% highlight python %}
select = FeatureSelectionInSample(max_log_k=7, n_folds=7, order_by_lmm=True, measure="ll", random_state=42)

best_k, feat_idx, best_mix, best_delta = select.run_select(G0.val, G0.val, y, cov=X_cov)    
{% endhighlight %}

`best_k` is the number of selected SNPs, `feat_idx` holds their indices, `best_mix` is $a_2$ in $K = (1-a_2)K_0 + a_2  K_1 $, and `best_delta` is $\delta$ in $\delta = \sigma^2 / \sigma^2_g$, the ratio of noise and genetic variance.

How these parameters are searched, is currently beyond my comprehension. As this is a high-level overview, I'll search for details later on.

- Then the $G_1$ is constructed and all the necessary parameters are fed to the underlying algorithm, `single_snp`, which calls the same algorithm internally as previously described `single_snp_leave_out_one_chrom`

{% highlight python %}
G1 = G0[:,feat_idx]
{% endhighlight %}

### Feature selection

Implemented in module
`fastlmm.feature_selection.feature_selection_two_kernel`

The boolean parameter `order_by_lmm` controls whether to compute P-values based on linear regression (`False`) or based on LMM computed with $G_0$ matrix (`True`).

Another parameter, `measure`, tells to the routine which statistic to use for the optimization, MSE or log-likelihood.

## In the next part: LMM(select) + PCs; epistasis; testing sets of SNPs
