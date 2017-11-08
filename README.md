---
title: "A manual for the use of the RCM functions"
author: "Stijn Hawinkel"
date: "November 08, 2017"
output: rmarkdown::github_document
---

# RCM

This repo contains R-code to fit and plot the RC(M)-models augmented with the negative binomial (which might be used in a later R-package) in the "R" folder. Moreover there is the "RC(M).Rmd" file with theoretical discussions, real data analyis and simulation results. The functions used for simulation but which are outside outside the core RC(M) algorithm are present in the "pubFun" folder.

The "ZIandSimCode.Rmd" file contains code to fit the ZI components as well and old simulation code.

## Manual

Here follows a short tutorial on fitting and plotting the RCM model.


```r
knitr::opts_chunk$set(cache = TRUE, autodep = TRUE, warning = FALSE, message = FALSE, 
    echo = TRUE, eval = TRUE, tidy = TRUE, fig.width = 9, fig.height = 6, purl = TRUE, 
    fig.show = "hold", cache.lazy = FALSE, fig.path = "README_figs/README-")
# The required package list:
reqpkg <- c("phyloseq", "MASS", "parallel", "nleqslv", "edgeR", "ggplot2", "alabama", 
    "reshape2", "tensor", "VGAM", "vegan")
# Load all required packages and show version
for (i in reqpkg) {
    print(i)
    print(packageVersion(i))
    library(i, quietly = TRUE, verbose = FALSE, warn.conflicts = FALSE, character.only = TRUE)
}
```

```
## [1] "phyloseq"
## [1] '1.20.0'
## [1] "MASS"
## [1] '7.3.47'
## [1] "parallel"
## [1] '3.4.2'
## [1] "nleqslv"
## [1] '3.2'
## [1] "edgeR"
## [1] '3.18.1'
## [1] "ggplot2"
## [1] '2.2.1'
## [1] "alabama"
## [1] '2015.3.1'
## [1] "reshape2"
## [1] '1.4.2'
## [1] "tensor"
## [1] '1.5'
## [1] "VGAM"
## [1] '1.0.3'
## [1] "vegan"
## [1] '2.4.3'
```

```r
palStore = palette()
funFiles = dir("R")
for (i in funFiles) {
    source(file.path("R", i))
}  # Run all fitting functions
```

## Dataset

As example data we use a study on the microbiome of colorectal cancer patients "Potential of fecal microbiota for early-stage detection of colorectal cancer" (2014) by Zeller _et al._.


```r
# Download the necessary files
for (i in c("data", "results")) {
    if (!dir.exists(i)) 
        dir.create(i)
}
download.file("http://users.ugent.be/~shawinke/data/species.txt", destfile = "./data/species.txt")
download.file("http://users.ugent.be/~shawinke/data/cancer_metadata.txt", destfile = "./data/cancer_metadata.txt")
zellerMeta = read.table("./data/cancer_metadata.txt", header = TRUE, sep = "")  #Read in the patient metadata
zellerS = read.table("./data/species.txt", header = TRUE, sep = "\t", row.names = 1)  #The 16S count data
```

Although our package does accept count matrices, we recommend merging all the data into a _phyloseq_ object fro RCM fitting.


```r
Sotu = otu_table(round(zellerS[-1, ]), taxa_are_rows = TRUE)  #Round to integers
Smeta = sample_data(zellerMeta)
rownames(Smeta) = gsub("-", ".", Smeta$Sample)
zellerSphy = phyloseq(Smeta, Sotu)  #Create the phyloseq object
```

## Unconstrained RCM

The unconstrained RC(M) method represents all variability present in the data, regardless of covariate information. It should be used as a first step in an exploratory analysis. We fit a model with two dimensions, which takes a few minutes.


```r
if (!file.exists(file = "./results/ZellerRCM2.RData")) {
    ZellerRCM2 = RCM(zellerSphy, k = 2)
    save(ZellerRCM2, file = "./results/ZellerRCM2.RData")
} else {
    load(file = "./results/ZellerRCM2.RData")
}
```

The runtime was 0.8 minutes to be exact. We plot the result of this ordination in a biplot, beginning with the samples.


```r
plot(ZellerRCM2, plotType = "samples")
```

![plot of chunk plotUnconstrainedRCMsam](README_figs/README-plotUnconstrainedRCMsam-1.png)

No clear signal is present at first sight. Note also that the plot is rectangular according to the values of the importance parameters $\psi$. In order to truthfully represent the distances between samples all axis must be on the same scale. We can add a colour code for the cancer diagnosis contained in the phyloseq object.


```r
plot(ZellerRCM2, plotType = "samples", samColour = "Diagnosis")
```

![plot of chunk plotUnconstrainedRCMsamCol](README_figs/README-plotUnconstrainedRCMsamCol-1.png)

It is clear that some of the variability of the samples is explained by the cancer status of the patients.

We can also add a richness measure as a colur, see ?phyloseq::estimate_richness for a list of available richness measures. Here we plot the Shannon diversity


```r
plot(ZellerRCM2, plotType = "samples", samColour = "Shannon")
```

![plot of chunk plotUnconstrainedRCMsamColShannon](README_figs/README-plotUnconstrainedRCMsamColShannon-1.png)

Next we plot only the species.


```r
plot(ZellerRCM2, plotType = "species")
```

![plot of chunk plotUnconstrainedRCMspec](README_figs/README-plotUnconstrainedRCMspec-1.png)

The researchers found that species from the Fusobacteria genus are associated with cancer. We can plot only these species using a regular expression.


```r
plot(ZellerRCM2, plotType = "species", taxRegExp = "Fusobacter", taxLabels = TRUE)
```

![plot of chunk plotUnconstrainedRCMspec2](README_figs/README-plotUnconstrainedRCMspec2-1.png)

It is clear that these Fusobacterium species behave very differently between the species.

Finally we can combine both plots into an interpretable biplot, which is the default for unconstrained RC(M). To avoid overplotting we only show the taxa with the 10 most important departures from independence.


```r
plot(ZellerRCM2, taxNum = 10, samColour = "Diagnosis")
```

![plot of chunk plotUnconstrainedRCMall](README_figs/README-plotUnconstrainedRCMall-1.png)

Samples are represented by dots, taxa by arrows. Both represent vectors with the origin as starting point.

Valid interpretatations are the following:

 - Samples (endpoints of sample vectors, the red dots) close together depart from independence in a similar way
 - The orthogonal projection of the taxon arrows on the sample arrows are proportional to the departure from independence of that taxon in that sample on the log scale, in the first two dimensions. For example Fusobacterium mortiferum is more abundant than average in samples on the left side of the plot, and more abundant in samples on the right side.
 - The importance parameters $\psi$ shown for every axis reflect the relative importance of the dimensions
 
Distances between endpoints of taxon vectors are meaningless.

We can also graphically highlight the departure from independence for a particular taxon and sample as follows:


```r
tmpPlot = plot(ZellerRCM2, taxNum = 10, samColour = "Diagnosis", returnCoords = TRUE)
addOrthProjection(tmpPlot, species = "Alloprevotella tannerae", sample = c(-1.2, 
    1.5))
```

![plot of chunk plotUnconstrainedRCMhighlight](README_figs/README-plotUnconstrainedRCMhighlight-1.png)

The projection of the species vector is graphically shown here, the orange bar representing the extent of the departure from independence. Note that we providid the exact taxon name, and approximate sample coordinates visually derived from the graph, but any combination of both is possible.

### Adding dimensions

Since the model is fitted dimension per dimension, we only have information on the first two dimension. If we want to add a third dimension, we do not have to start from scratch but can use the previously fitted model in two dimension as a starting point and additionally estimate the third dimension.


```r
if (!file.exists(file = "./results/ZellerRCM3.RData")) {
    ZellerRCM3 = RCM(zellerSphy, prevFit = ZellerRCM2, k = 3)
    save(ZellerRCM3, file = "./results/ZellerRCM3.RData")
} else {
    load(file = "./results/ZellerRCM3.RData")
}
```

The total runtime for all dimensions combined was 2.3. We can then plot any combination of two dimensions we want, e.g. the first and the third.


```r
plot(ZellerRCM3, Dim = c(1, 3), samColour = "Diagnosis", taxNum = 6)
```

![plot of chunk plotAddedDimension](README_figs/README-plotAddedDimension-1.png)

The third dimension also correlates with a separation of cancer patients vs. healthy and small adenoma patients.

### Assessing the goodness of fit

Some taxa (or samples) may not follow a negative binomial distribution, or their departures from independence may not be appropriately represented in low dimensions. We visualize these taxa and samples through their deviances, which are the squared sums of their deviance residuals. This allows us to graphically identify taxa and samples that are poorly represented in the current ordination.

The deviance residuals can be extracted manually


```r
devResiduals = getDevianceRes(ZellerRCM2, Dim = c(1, 2))
```

but you can also just supply the argument "Deviance" to samColour


```r
plot(ZellerRCM2, plotType = "samples", samColour = "Deviance", samSize = 2.5)
```

![plot of chunk plotUnconstrainedRCMsamColDev](README_figs/README-plotUnconstrainedRCMsamColDev-1.png)

Samples with the largest scores exhibit the poorest fit. This may indicate that samples with strong departures from independence acquire large scores, but still are not well represented in lower dimensions. Especially the one bottom right may be a problematic case.

We again also use the third dimension, maybe this sample is fitted well there?


```r
plot(ZellerRCM3, plotType = "samples", samColour = "Deviance", samSize = 2.5, 
    Dim = c(1, 3))
```

![plot of chunk plotUnconstrainedRCMsamColDev13](README_figs/README-plotUnconstrainedRCMsamColDev13-1.png)

No the problem persists there.

The same principle can be applied to the taxa


```r
plot(ZellerRCM3, plotType = "species", taxCol = "Deviance", samSize = 2.5, Dim = c(1, 
    2), arrowSize = 0.5)
```

![plot of chunk plotUnconstrainedRCMtaxDev](README_figs/README-plotUnconstrainedRCMtaxDev-1.png)

For the taxa it appears to be the taxa with smaller scores are the more poorly fitted ones. Note that since the count table is not square, we cannot compare sample and taxon deviances. They have not been calculated based on the same number of taxa. Also, one cannot do chi-squared tests based on the deviances since this is not a classical regression model, but an overparametrized one.

## Constrained analysis

In this second step we look for the variability in the dataset explained by linear combinations of covariates that maximally separate the niches of the species. This should be done in a second step, and preferably only with variables that are believed to have an impact on the species' abundances. Here we used the variables age, gender, BMI, country and diagnosis in the gradient.
In this analysis all covariates values of a sample are projected onto a single scalar, the environmental score of this sample. The projection vector is called the environmental gradient, the magnitude of its components reveals the importance of each variable. The taxon-wise response functions then describe how the logged mean abundance depends on the environmental score.

### Linear response functions

Even though these response functions may be too simplistic, they have the advantage of being easy to interpret (and plot).


```r
if (!file.exists(file = "./results/ZellerRCM2constr.RData")) {
    ZellerRCM2constr = RCM(zellerSphy, k = 2, covariates = c("Age", "Gender", 
        "BMI", "Country", "Diagnosis"), responseFun = "linear")
    save(ZellerRCM2constr, file = "./results/ZellerRCM2constr.RData")
} else {
    load(file = "./results/ZellerRCM2constr.RData")
}
```

First we plot the samples


```r
plot(ZellerRCM2constr, plotType = c("samples"))
```

![plot of chunk constrLinPlot](README_figs/README-constrLinPlot-1.png)

In the constrained analysis we clearly see three groups of samples appearing.


```r
plot(ZellerRCM2constr, plotType = c("samples"), samColour = "Diagnosis")
```

![plot of chunk constrLinPlot2](README_figs/README-constrLinPlot2-1.png)

One group are the healthy patients, the other two are cancer patients.


```r
plot(ZellerRCM2constr, plotType = c("samples"), samColour = "Country")
```

![plot of chunk constrLinPlot3](README_figs/README-constrLinPlot3-1.png)

The cancer patients are separated by country. Note that from Germany there are only cancer patients in this dataset.

Now we add the species to make a biplot


```r
plot(ZellerRCM2constr, plotType = c("species", "samples"))
```

![plot of chunk plotlin2cor](README_figs/README-plotlin2cor-1.png)

The interpretation is similar as before: the orthogonal projection of a taxon's arrow on a sample represents the departure from independence for that taxon in that sample, _explained by environmental variables_. New is also that the taxa arrows do not start from the origin, but all have their own starting point. This starting point represents the environmental scores for which there is no departure from independence. The direction of the arrow then represents how its expected abundance increases. Again we can show this visually:


```r
tmpPlot2 = plot(ZellerRCM2constr, plotType = c("species", "samples"), returnCoords = TRUE)
addOrthProjection(tmpPlot2, species = "Pseudomonas fluorescens", sample = c(-12, 
    7))
```

![plot of chunk plotlin2corVis](README_figs/README-plotlin2corVis-1.png)

Note that the projection bar does not start from the origin in this case either.

Next we can make a biplot of taxa and environmental variabless.


```r
plot(ZellerRCM2constr, plotType = c("species", "variables"))
```

![plot of chunk plotlin3](README_figs/README-plotlin3-1.png)

The projection of species arrows on environmental variables (starting from the origin) represents the sensitivity of this taxon to changes in this variables. Note that the fact that the arrows of BMI and gender are of similar length indicates that one _standard deviation_ in BMI has a similar effect to gender.

Also this interpretation we can show visually on the graph


```r
tmpPlot3 = plot(ZellerRCM2constr, plotType = c("species", "variables"), returnCoords = TRUE)
addOrthProjection(tmpPlot3, species = "Pseudomonas fluorescens", variable = "DiagnosisSmall_adenoma")
```

![plot of chunk plotlin3Vis](README_figs/README-plotlin3Vis-1.png)

We observe that country and diagnosis are the main drivers of the environmental gradient. We also see that healthy patients are very similar to small adenoma patients, but that they are very different from cancer patients.

To finish we also show the triplot, which unites all the information of the ordination:


```r
plot(ZellerRCM2constr)
```

![plot of chunk plotlin3Triplot](README_figs/README-plotlin3Triplot-1.png)

Note that the samples and the environmental variables cannot be related to each other.

#### Assessing the goodness of fit

Also for constrained ordination it can be interesting to use the deviance residuals. We can use them in the unconstrained case by summing over taxa or samples, or plot them versus the environmental gradient to detect lack of fit for the shape of the response function. For this latter goal we provide two procedures: a diagnostic plot of the taxa with the strongest response to the environmental gradient, or an automatic trend detection using the runs test statistic by Ward and Wolfowitz. Checking the linearity (or Gaussian) assumption of the response function is crucial: excessive departure invalidate the interpretation of the ordination plot.

A deviance residual plot for the strongest responders


```r
residualPlot(ZellerRCM2constr, whichTaxa = "response")
```

![plot of chunk plotDevResp](README_figs/README-plotDevResp-1.png)

The same taxa but with Pearson residuals 


```r
residualPlot(ZellerRCM2constr, whichTaxa = "response", resid = "Pearson")
```

![plot of chunk plotPearResp](README_figs/README-plotPearResp-1.png)

Most species do not exhibit any obvious pattern, although departures seem to increase with environmental scores.

The runs test is a test that automatically attempts to detect non randomness in the sequence of positive and negative residuals. We plot the residual plots for the taxa with the largest run statistic (since the sample sizes and null distributions are equal this corresponds with the smallest p-values, but we do not do inference here because of the non-classical framework). 


```r
residualPlot(ZellerRCM2constr, whichTaxa = "runs", resid = "Deviance")
```

![plot of chunk plotDevRuns](README_figs/README-plotDevRuns-1.png)


```r
residualPlot(ZellerRCM2constr, whichTaxa = "runs", resid = "Pearson")
```

![plot of chunk plotPearRuns](README_figs/README-plotPearRuns-1.png)

For these taxa we see no signal at all.

#### Identifying influential observations

It can also be interesting to see which samples have the strongest influence on the environmental gradient.


```r
inflArr = NBalphaInfl(ZellerRCM2constr, Dim = 1)
```

Say we want to know which samples have on average the largest influence on the estimation of the age parameter.


```r
plot(ZellerRCM2constr, plotType = c("variables", "samples"), samColour = rowSums(inflArr[, 
    , "Age"]), colLegend = "Influence on Age parameter in dimension 1")
```

![plot of chunk inflAge](README_figs/README-inflAge-1.png)

Below we see a couple of very influential observations. They may be very young?


```r
plot(ZellerRCM2constr, plotType = c("variables", "samples"), samColour = "Age")
```

![plot of chunk inflAgeCol](README_figs/README-inflAgeCol-1.png)

Indeed, a few young subjects affect the estimation of the age parameter most. One should always be wary of these kind of "outliers" that strongly affect the ordination.

We can achieve the same plots by using the "influence" flag in the plotting function. 


```r
plot(ZellerRCM2constr, plotType = c("variables", "samples"), samColour = "Age", 
    Influence = TRUE)
```

![plot of chunk inflAgeFast](README_figs/README-inflAgeFast-1.png)

The age coefficient is largest in the second dimension actually, let's look at the influence on that component


```r
plot(ZellerRCM2constr, plotType = c("variables", "samples"), samColour = "Age", 
    Influence = TRUE, inflDim = 2)
```

![plot of chunk inflAgeFast2](README_figs/README-inflAgeFast2-1.png)

One last illustration: which samples have the strongest impact on the "DiagnosisCancer" parameter


```r
plot(ZellerRCM2constr, plotType = c("variables", "samples"), samColour = "DiagnosisCancer", 
    Influence = TRUE, samShape = "Diagnosis", samSize = 2)
```

![plot of chunk inflDiagFast](README_figs/README-inflDiagFast-1.png)

French cancer patients appear to have the strongest impact.

### Non-parametric response functions

These response functions are very data driven, but have the drawback that they do not allow to visualize the role of the taxa in the ordination.


```r
if (!file.exists(file = "./results/ZellerRCM2constrnonParam.RData")) {
    ZellerRCM2constrNonParam = RCM(zellerSphy, k = 2, covariates = c("Age", 
        "Gender", "BMI", "Country", "Diagnosis"), responseFun = "nonparametric")
    save(ZellerRCM2constrNonParam, file = "./results/ZellerRCM2constrNonParam.RData")
} else {
    load(file = "./results/ZellerRCM2constrNonParam.RData")
}
```


```r
plot(ZellerRCM2constrNonParam, plotType = "samples")
```

![plot of chunk plotnonParam](README_figs/README-plotnonParam-1.png)

Using non-parametric response functions we do not find the same clear clusters


```r
plot(ZellerRCM2constrNonParam, plotType = "samples", samColour = "Diagnosis")
```

![plot of chunk plotnonParamCol](README_figs/README-plotnonParamCol-1.png)

Show the variables


```r
plot(ZellerRCM2constrNonParam, plotType = "variables")
```

![plot of chunk plotnonParamCol3](README_figs/README-plotnonParamCol3-1.png)

The envrionmental gradients are quite different from the case with the linear response functions. Especially age is an important driver of the environmental gradient here. Still the results for cancer diagnosis , country and gender are similar to before.
