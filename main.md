---
title: "scran.js: single-cell data analysis in the browser"
author:
  - name: Aaron Lun
    email: infinite.monkeys.with.keyboards@gmail.com
  - name: Jayaram Kancherla
    email: jayaram.kancherla@gmail.com
---

# Introduction

We do it. 
You do it. 
Everyone does it.
No, it's not cocaine, we're talking about single-cell RNA sequencing (scRNA-seq).
As its name suggests, this technology enables us to study the gene expression of cell populations at the level of the constituent cells.
This enables researchers to easily characterize cell subpopulations without requiring any prior knowledge.
No longer do we have to painstakingly purify each subpopulation for RNA sequencing of bulk populations;
instead, we can effectively perform the purification _in silico_, in a type of "super flow cytometry" that is the bread and butter of scRNA-seq data analysis.

One might expect that the computational analysis of scRNA-seq data would be critical for its interpretation.
(Thereby justifying cushy six-figure salaries for computational biologists around the world.)
A typical analysis involves a number of routine steps - 
removal of low-quality cells, normalization for cell-specific biases, selection of interesting features, dimensionality reduction, clustering and visualization.
The goal is to quantify the heterogeneity of the input population, by describing it in terms of a small number of discrete subpopulations.
These subpopulations serve as the starting point for further interrogation of the data to address specific hypotheses.

While it sounds simple, the definition of these subpopulations is very much a "choose your own adventure" process.
One can fiddle with parameters at every step of the process to change how the cells are clustered together. 
Practitioners will usually perform several iterations of the analysis to explore the dataset from different perspectives.
As with politics, philosophy, and leaving the toilet seat up, there is no right or wrong view; but some are more useful than others, and some trial and error is often required to find them.
This drives the dominance of interactive data analysis environments such as R and Python with their associated software suites for scRNA-seq data analysis.

Of course, both R and Python are programming languages that involve a modest-to-steep learning curve depending on one's aptitude.
This is a tough ask for bench biologists, many of whom are expected to analyze their own datasets despite the lack of training and experience.
(Interestingly, the converse is not true; computational biologists are never expected to generate their own data, and indeed, one of us is not allowed near electron microscopes anymore.)
To this end, a number of graphical user interfaces (GUIs) have been developed that provide an intuitive "point-and-click" approach to scRNA-seq data analysis.
Users can easily upload their data, adjust parameters and visualize results without the need for prior programming knowledge, thus democratizing the analysis to a wider audience of potential analysts.

One aspect of existing scRNA-seq GUIs is that they are primarily interfaces to a backend server that uses R or Python to perform the actual analysis.
This is a pragmatic design that re-uses existing analysis software but introduces a number of additional complexities.
Deployment of the application is more problematic as the backend must be configured and kept available for the lifetime of the application.
This is not necessarily easy or cheap, especially if the application needs to scale to an arbitrarily large number of users.
The need for constant communication with the server introduces extra latency that can cause the interactive experience to deteriorate.
Questions of data ownership and privacy also arise when data is transferred to someone else's server, potentially leading to bonus-deflating fines under regulations like the CCPA and GDPR.

Our solution to all these problems is to simply create a web application that performs the analysis directly on the user's computer. 
This eliminates the need for a computing backend while still retaining a intuitive GUI that requires no programming experience to set up or operate.
In this article, we demonstrate how to achieve analysis client-side compute with **scran.js**, a web application and Javascript library for scRNA-seq data analysis in the browser.
We take advantage of the WebAssembly standard to efficiently execute routine analysis steps, combined with established Javascript frameworks for interactive visualization of the results.
Our application can complete a standard analysis of a 68,000-cell dataset in under two minutes on unremarkable hardware, encouraging comprehensive inspection of the data by enabling rapid iteration.
We believe **scran.js** serves as a paradigm-nudging illustration of the feasibility of the browser as a compute engine for bioinformatics data analysis.

# Application overview

**scran.js** provides a simple linear interface that guides the user through a routine scRNA-seq analysis.

1. We import a gene-by-cell count matrix from the user's machine, typically in the form of Matrix Market files such as those produced by the Cellranger pipeline.
_HDF5 files are also supported._
2. We compute common quality control (QC) metrics such as the total count, number of detected genes and proportion of mitochondrial counts.
Low-quality cells are then defined as those cells with outlier values for any of these metrics and are removed.
3. We perform scaling normalization based on the library size to remove cell-specific biases.
This is followed by a log-transformation to obtain log-normalized expression values.
4. We fit a trend to the per-gene variances with respect to the means, computed from the log-expression vaules.
We sort on the residuals to define a subset of highly variable genes (HVGs). 
5. We then perform principal components analysis (PCA) on the HVG subset of the log-expression.
This yields a few top PCs that capture the heterogeneity of the data in a compressed and denoised form.
6. We apply a variety of clustering techniques on the top PCs to generate discrete subpopulations.
Each cluster is characterized by performing differential expression analyses to detect its top markers.
7. Most importantly, we generate our Figure 1 by performing further dimensionality reduction on the top PCs.
This includes the usual t-distributed stochastic neighbor embedding (t-SNE) and uniform manifold approximation and projection (UMAP).

The statistical and scientific rationale behind each of these steps is discussed in more detail in the [OSCA book](https://bioconductor.org/books/release/OSCA/) and we shall not discuss it further here.
