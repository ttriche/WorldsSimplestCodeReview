---
title: "Scaling tests of proportions up and down"
author: "Tim Triche"
date: "December 2nd, 2021"
output: 
  html_document:
    keep_md: true
vignette: >
  %\VignetteIndexEntry{proportionsAndPower}
  %\VignetteEngine{knitr::rmarkdown}
  \usepackage[utf8]{inputenc}

---



# Statistical power for tests of proportions: WT1 and TET2

The gist: Extract a table of results from a PDF and then perform tests on it. 


## The tabulizer package and its discontents 

[tabulizer](https://cran.r-hub.io/web/packages/tabulizer/) is great, 
but currently it's in the [penalty box](https://cran.r-project.org/package=tabulizer) on [CRAN](https://CRAN.R-project.org/) (where most packages for R usually
live). So, until the version on CRAN is fixed, you can try something like this 
if you want to use `tabulizer` in your code:

<details>
  <summary>Click to show tabulizer install instructions</summary> 

```r

# {{{ Don't run this unless you need to 
if (FALSE) {

  # installing packages from within vignettes is bad,
  # so we don't execute this chunk and we fence it off 
  if (!require("remotes")) install.packages("remotes")
  if (!require("BiocManager")) install.packages("BiocManager")
  BiocManger::install(c("ropensci/tabulizerjars", "ropensci/tabulizer"))

} 
# }}}

```
</details> 

In the following, I'll illustrate how I used this to extend past last week's 
IDH/TET mutual exclusivity testing in AML (a liquid tumor) to investigate the 
larger sample size required for a similar exercise in solid tumors, and for a
test with more degrees of freedom (IDH1 vs IDH2 vs TET2 vs WT1) that offered
novel biological insights into a recurrent mutation in Wilms tumor and AML.

## Data extraction for reproducing results and performing power analysis

Let's use the raw data from [this 2014 Molecular Cell paper](https://doi.org/10.1016/j.molcel.2014.12.023) to look at what you can do with bigger sample sizes.

### Liquid tumors

Some of the data is in the paper, and some of it is in the supplement. We can 
automate extraction of either or both. The authors note their meta-analysis of 
six different AML studies, where 303 out of 1057 cases (28.7%) carried mutations
in _IDH1, IDH2, TET2_, and/or _WT1_. Eyeballing panel B of figure 1, we find:

* 754 samples (1057 - 303) were wild-type for all four genes,
* 85 samples carry a _TET2_ mutation,
* 84 samples carry an _IDH2_ mutation,
* 66 samples carry an _IDH1_ mutation,
* 56 samples carry a _WT1_ mutation, 
* 4 samples carry _WT1_ and _TET2_ mutations,
* 3 samples carry _IDH1_ and _TET2_ mutations,
* 2 samples carry _IDH1_ and _IDH2_ mutations,
* 2 samples carry _IDH1_ and _WT1_ mutations,
* 1 sample carries _IDH2_ and _WT1_ mutations,
* no samples carry three of the mutations, and
* no samples carry all four mutations.

In other words, we can expand the data back out like so. (There's probably a 
tidy way to do this, but I don't know it. Bonus points to anyone who does.) 


```r

library(tibble) 

# need to do a bit of munging to start out... 
rows <- c("TET2", "IDH2", "IDH1", "WT1", "count")
combos <- list( idh1wt1 = c(0, 0, 1, 1, 2),
                idh1 = c(0, 0, 1, 0, 66),
                idh1tet2 = c(1, 0, 1, 0, 3),
                idh1idh2 = c(0, 1, 1, 0, 2),
                idh2 = c(0, 1, 0, 0, 84),
                idh2wt1 = c(0, 1, 0, 1, 1),
                wt1 = c(0, 0, 0, 1, 56),
                wt1tet2 = c(1, 0, 0, 1, 4),
                tet2 = c(1, 0, 0, 0, 85), 
                none = c(0, 0, 0, 0, 754) )
design <- data.frame(do.call(cbind, lapply(combos, as.integer)))
rownames(design) <- rows  

# data(design, package="WorldsSimplestCodeReview")

# a data.frame is a list of vectors, so we can just apply a function to each.
# here, we'd like a tibble that has `counts` rows of the right indicators.
expand <- function(x) {
  result <- t(replicate(x[which(rows == "count")], x[which(rows != "count")]))
  colnames(result) <- rows[1:4] # i.e., the gene names
  tibble(data.frame(result)) 
}

# Reduce(fn, list) repeatedly applies `fn` to `list` to yield a result:
Reduce(add_row, lapply(design, expand)) %>% 
  select(IDH1, IDH2, WT1, TET2) -> 
    bigtbl

glimpse(bigtbl) 
#> Rows: 1,057
#> Columns: 4
#> $ IDH1 <int> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,…
#> $ IDH2 <int> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,…
#> $ WT1  <int> 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,…
#> $ TET2 <int> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,…
colSums(bigtbl)
#> IDH1 IDH2  WT1 TET2 
#>   73   87   63   92

# data(bigtbl, package="WorldsSimplestCodeReview")
```

Suppose we wanted to check our work.  We could generate something similar to 
panel A in figure 1 using the `pheatmap` package (which isn't super tidy, but
them's the breaks): 


```r

library(pheatmap) 
bigtbl %>% t %>% # transpose
  pheatmap(legend=FALSE, 
           cluster_cols=FALSE,
           cluster_rows=FALSE, 
           col=c("white", "darkred"), 
           main="Mutations in 1057 AML cases")
```

![plot of chunk pheatmap](figure/pheatmap-1.png)

That looks about right (granted, we made it a bit easier to see the overlaps). 
It turns out that we can frame this several ways in terms of statistical tests. 
The authors don't bother, but let's run with the theory that _IDH1/2_ and _WT1_
mutations both lead to (among other things) a loss of targeted _TET2_ activity
at critically important regions of the genome used in myeloid differentiation.
We'll perform a contingency test (_IDH_ & _WT1_ vs _TET2_) to evaluate it, in
no small part because we know there aren't any triple-hit cases to worry about.


```r

# for tabyl() 
library(janitor) 

# tidy up
bigtbl %>% 
  mutate(IDH = as.integer(IDH1 | IDH2)) %>% 
  mutate(IDH_WT1 = case_when(IDH == 1 & WT1 == 1 ~ "both", 
                             IDH == 1            ~ "mIDH",
                             WT1 == 1            ~ "mWT1",
                             TRUE                ~ "wt"))    %>% 
  mutate(TET =     case_when(TET2 == 1           ~ "mTET2", 
                             TRUE                ~ "wt"))    %>% 
  tabyl(IDH_WT1, TET) -> 
    testtabyl

# quick look
testtabyl
#>  IDH_WT1 mTET2  wt
#>     both     0   3
#>     mIDH     3 152
#>     mWT1     4  56
#>       wt    85 754
```

Even though the authors didn't test these comparisons, they are significant: 


```r

# getElement is similar to `[[` 
testtabyl %>% 
  fisher.test %>% 
  getElement("p.value") -> 
    fet.p

testtabyl %>% 
  chisq.test(simulate.p.value = TRUE) %>% 
  getElement("p.value") -> 
    chisq.p

# p.adjust corrects for multiple comparisons, which we can discuss more
method <- c("none", "fdr", "holm", "bonferroni")
names(method) <- method 
pvals <- c(fisher = fet.p, chisq = chisq.p)
adjusted_p <- data.frame(lapply(method, 
                                function(m) p.adjust(pvals, method=m)))

# for clarity
t(adjusted_p) 
#>                 fisher      chisq
#> none       0.002776113 0.02898551
#> fdr        0.005552226 0.02898551
#> holm       0.005552226 0.02898551
#> bonferroni 0.005552226 0.05797101
```

_Question:_ What does `simulate.p.value` do? What happens if you omit it? 

We can see that the alpha level for our combination of two test results 
changes when we adjust for the fact that we ran two separate tests. How 
much it changes depends on the way we correct for multiple testing, and 
it turns out that there is a hierarchy in terms of stringency. Roughly, 

* `none`      : the raw p-values, as if we'd done just one test. 
* `fdr`       : the p-values after applying Benjamini-Hochberg FDR correction.
* `holm`      : the p-values after appling Holm's FWER correction.
* `bonferroni`: the p-values after applying Bonferroni correction. 

Benjamini & Hochberg's step-up FDR (false discovery rate) procedure controls 
the aggregate false positive rate so that the rank-ordered (smallest to largest)
adjusted p-values can be compared against a threshold for significance and the
odds of any particular comparison being wrong will be no greater than _pBH_. 
In other words, BH controls odds of an individual test being a false positive.

Holm, on the other hand, controls the FWER (family-wise error rate) so that the 
the probability of _at least one_ false positive result is bounded by _pHolm_. 
In other words, Holm's method controls odds of any test being a false positive. 

_Question:_
Why does the result for `chisq.p` change so much when Bonferroni correction
is applied (as opposed to Holm FWER, or Benjamini-Hochberg FDR, correction)? 
Keep in mind that _both_ Holm's and Bonferroni's methods control the FWER. 

<details>
  <summary>Click for answer</summary>

Bonferroni's method is universally less powerful than Holm's method because,
once we have rejected one of a set of _m_ null hypotheses, there are only _m_-1
hypotheses left to test, and thus dividing our nominal significance threshold 
by the original _m_ does not make any sense. The only time it makes sense to 
use Bonferroni's method is when dealing with unsophisticated reviewers, as when
one wishes to secure a larger budget than is necessary for an experiment, since
almost all grants are cut by some percentage. In this case, one uses the worst
bids for every reagent and service, along with the most ridiculous multiple 
comparisons correction, to give the appearance of rigor (and actually have 
enough money to conduct a rigorous experiment as designed, if it comes to pass). 
</details> 

### Solid tumors

The authors also note that the sample size for solid tumors in which 
_WT1_ and/or _TET2_ are mutated is too small for statistical analysis. 
We shall evaluate this claim below. (Spoiler: it depends on your alpha)

### Rounding up the data 

We will use a few packages for this exercise; make sure they're installed first.
Note that you can use additional packages for analytic power calculations if 
you wish, or you can use resampling. I don't really care which one you choose,
but if you learn to simulate via resampling, you won't have to search for code 
to do the magic when you have weird or complicated models to compare later on.
For now, let's just use built-in functions unless they're too obnoxious (e.g.
`base::table`, `base::chisq.test`, and `base::fisher.test`) to use tidily. 


```r

library(tabulizer)  # for extracting tables from PDFs; see note above re:this 
library(magrittr)   # for `%<>%` which means "replace the source w/the result"
library(janitor)    # for easier 2x2 testing 
library(tibble)     # for manipulation
library(dplyr)      # for the usual 
```

It turns out that once `tabulizer` is installed, we can fetch data remotely 
same as if we pulled it from a local PDF file. This can be handy when we want
to provide examples that work across various peoples' installations. Of course,
the drawback can be that a library won't install, so I've saved the result to
the data directory for the package and left the demonstration as an exercise. 

Adapting [the tabulizer vignette](https://cran.r-hub.io/web/packages/tabulizer/vignettes/tabulizer.html). We will not execute the next chunk because I can't 
guarantee that both R and the Java libraries for `tabula` will work for you. 

<details>
  <summary> How the data gets extracted from the supplement </summary>


```r

# {{{
# for `%<>%`
library(magrittr)

# clean output below:
cleanup_df <- function(x) { 
  colnames(x) <- x[1, ]
  keeprows <- setdiff(which(x[, 5] != ""), 1)
  tbl <- tibble(as.data.frame(x[keeprows, ]), .name_repair="universal")
  for (i in c("Patient", "WT1", "TET2", "Co.occurrence")) {
    tbl[[i]] %<>% as.integer
  }
  return(tbl) 
}

# Table S2:
pages <- 17

# local
f <- system.file("examples", 
                 "Supplement_Roles_of_mutant_IDH1_IDH2_TET2_and_WT1_in_AML.pdf",
                 package="WorldsSimplestCodeReview")

# on the web
furl <- "https://www.cell.com/cms/10.1016/j.molcel.2014.12.023/attachment/99ebef13-5dc8-4efd-8e2a-b8aa115647df/mmc1.pdf"


# We will skip most of the processing code because I can't guarantee
# that tabulizer's Java code will run on your install (see previous). 
if (FALSE) { # i.e., "don't run this code" 

  library(tabulizer)
  if (file.exists(f)) tbl0 <- extract_tables(f, pages=pages)[[1]]
  tbl <- extract_tables(furl, pages=pages)[[1]]

  # Oh god that's ugly. Let's apply that cleanup function from before.  
  extract_tables(furl, pages=pages)[[1]] %>% cleanup_df -> solidtumors
  glimpse(solidtumors) 


}
# }}}

```
</details>

So, you don't have to trust me (above), but you also don't have to suffer
through getting `tabulizer` installed. Seems fair, no? Let's use the results.


```r

data(solidtumors, package="WorldsSimplestCodeReview")

# for tibbling 
make2x2 <- function(x) {
  matrix(as.integer(x[, c("neither", "tet2only", "wt1only", "both")]), nrow=2)
}
fisher <- function(x) fisher.test(make2x2(x))$p.value
chisq <- function(x) chisq.test(make2x2(x), simulate.p=TRUE)$p.value

# for row-wise testing 
library(purrrlyr) 

# tests
solidtumors %>% 
  rename(total = Patient) %>% 
  rename(both = Co.occurrence) %>% 
  mutate(tet2only = TET2 - both) %>% 
  mutate(wt1only = WT1 - both) %>% 
  mutate(either = wt1only + tet2only + both) %>% 
  mutate(neither = total - either) %>% 
  mutate(cancer = make.unique(Cancer.types)) %>% 
  select(cancer, neither, tet2only, wt1only, both, Reference) %>% 
  by_row(fisher, .to="fet.p", .collate="cols") %>% 
  by_row(chisq, .to="chisq.p", .collate="cols") %>% 
  arrange(fet.p) -> 
    test_results

# test across all the solid tumors above combined
test_results %>% 
  select(neither, tet2only, wt1only, both) %>% 
  colSums %>% 
  t %>% 
  data.frame %>%
  tibble %>% 
  mutate(cancer = "overall") %>% 
  mutate(Reference = NA) %>% 
  by_row(fisher, .to="fet.p", .collate="cols") %>% 
  by_row(chisq, .to="chisq.p", .collate="cols") %>% 
  select(colnames(test_results)) ->
    overall
```

_Question:_ What happens if you correct for multiple comparisons above? 


```r

test_results %>%
  add_row(overall, .before=1) %>% 
  select(cancer, fet.p, chisq.p) %>% 
  mutate(fet.fdr = p.adjust(fet.p, method="fdr")) %>% 
  mutate(fet.holm = p.adjust(fet.p, method="holm")) %>% 
  mutate(chisq.fdr = p.adjust(chisq.p, method="fdr")) %>%  
  mutate(chisq.holm = p.adjust(chisq.p, method="holm")) 
#> # A tibble: 18 × 7
#>    cancer                    fet.p chisq.p fet.fdr fet.holm chisq.fdr chisq.holm
#>    <chr>                     <dbl>   <dbl>   <dbl>    <dbl>     <dbl>      <dbl>
#>  1 overall                 0.0359  0.0410   0.323    0.611     0.369      0.697 
#>  2 Colorectal.1            0.00352 0.00150  0.0633   0.0633    0.0270     0.0270
#>  3 Stomach                 0.0716  0.0685   0.429    1         0.411      1     
#>  4 Colorectal              0.197   0.219    0.885    1         0.987      1     
#>  5 Bladder Urothelial      1       1        1        1         1          1     
#>  6 Lung Adenocarcinoma.1   1       1        1        1         1          1     
#>  7 Breast Invasive         1       1        1        1         1          1     
#>  8 Kidney Renal Clear Cell 1       1        1        1         1          1     
#>  9 Kidney Renal Papillary  1       1        1        1         1          1     
#> 10 Liver Hepatocellular    1       1        1        1         1          1     
#> 11 Lung Adenocarcinoma     1       1        1        1         1          1     
#> 12 Small Cell Lung Cancer  1       1        1        1         1          1     
#> 13 Multiple Myeloma        1       1        1        1         1          1     
#> 14 Skin Cutaneous          1       1        1        1         1          1     
#> 15 Skin Cutaneous.1        1       1        1        1         1          1     
#> 16 Uterine Corpus          1       1        1        1         1          1     
#> 17 Lung Squamous Cell      1       1        1        1         1          1     
#> 18 Skin Cutaneous.2        1       1        1        1         1          1
```

_Question:_ Can you add Bonferroni-corrected values to the above?  Should you?

