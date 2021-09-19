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
No, not cocaine; we're talking about single-cell RNA sequencing (scRNA-seq).
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

The statistical and scientific rationale behind each of these steps is discussed in more detail in the [OSCA book](https://bioconductor.org/books/release/OSCA/).

At each step, **scran.js** allows users to easily customize key parameters.
For example, we can adjust the thresholds used for the QC filters, the number of HVGs and top PCs, the granularity of the clustering, and more.
_When the parameters are modified for any step, all subsequent steps are easily re-executed to determine the effect of the change._
_Customized parameters can be saved into a URL bookmark that can be used to restore an existing custom analysis._
_Alternatively, the entire customized analysis and its results can be exported to a set of static files (HTML and CSVs) for further sharing._

Where relevant, each step provides an appropriate interactive visualization of the results.
QC metric distributions are visualized with violin plots where the thresholds can be manually adjusted by moving the filter up or down.
_The mean-variance trend is displayed on a scatter plot where genes of interest can be highlighed for closer inspection on a linked table._
The t-SNE and UMAP plots are animated and show the evolution of the embedding as it is computed across iterations.
These plots also support brushing to define custom subpopulations for further use in marker detection.

# Efficient compute with WebAssembly

WebAssembly (Wasm) is a standard instruction format that serves as a web-executable compilation target for higher-level languages like C/C++, Go and Rust.
In other words, code written in a language like C++ can be compiled to Wasm and embedded inside a web application for execution by the browser at near-native performance. 
This provides an opportunity to turn the browser into a compute engine by integrating existing scientific libraries for bioinformatics data analysis.
Indeed, we were greatly inspired by the [**biowasm**](https://github.com/biowasm) project, which provides Wasm binaries for common genomics tools like `samtools` and `bedtools`. 
This inspiration was not so much from the code itself, but from the hope that it offered: after all, if someone managed to compile HTSlib to Wasm, it should be possible to compile anything to Wasm.

To this end, we collected C++ implementations of the algorithms required for each analysis step.
Most of the code was harvested from existing R packages, with varying levels of difficulty to convert them into a pure C++ library.
A list of the relevant libaries and their origins follows:

- The [**tatami**](https://github.com/LTLA/tatami) library provides an abstract interface to different matrix classes, focusing on row and column extraction.
While it supports the usual dense and sparse representations, its key feature is its support for delayed operations,
mimicking the [**DelayedArray**](https://bioconductor.org/packages/DelayedArray) Bioconductor package.
This allows the creation of log-normalized matrices and HVG submatrices with no extra memory usage.
- The [**knncolle**](https://github.com/LTLA/knncolle) library wraps a number of nearest neighbor detection methods in a consistent interface.
This includes exact methods like vantage point tree search as well as approximate methods like [**Annoy**](https://github.com/spotify/Annoy).
It is an equivalent to the [**BiocNeighbors**](https://github.com/LTLA/BiocNeighbors) Bioconductor package.
- The [**CppIrlba**](https://github.com/LTLA/CppIrlba) library contains a C++ port of the IRLBA algorithm for approximate PCA,
enabling us to obtain the top few PCs in a highly efficient manner.
The code here was ported from the [**irlba**](https://github.com/bwlewis/irlba) package, with some refactoring to eliminate dependencies on R-specific libraries.
- The [**CppKmeans**](https://github.com/LTLA/CppKmeans) library contains C++ implementations of the Hartigan-Wong and Lloyd algorithms for k-means clustering.
In particular, the Hartigan-Wong implementation was translated from the Fortran code used by R's `kmeans` function.
- The [**CppWeightedLowess**](https://github.com/LTLA/CppWeightedLowess) package contains a C++ implementation of the `weightedLowess` function in the [**limma**](https://bioconductor.org/packages/limma) package.
This is a variant of the standard LOWESS implementation from R's `lowess` function, modified so that the weights are allowed to influence both the span calculation and the linear regression.
- The [**qdtsne**](https://github.com/LTLA/qdtsne) library contains an implementation of the Barnes-Hut t-SNE algorithm.
This is mostly a refactored version of the code in the [**Rtsne**](https://cran.r-project.org/web/packages/Rtsne/index.html) package, itself refactored from the code provided with the original paper.
Some additional optimizations have been applied to improve scalability.
- The [**umappp**](https://github.com/LTLA/umappp) library provides an implementation of the UMAP dimensionality reduction algorithm.
This is largely derived from code in the [**uwot**](https://cran.r-project.org/web/packages/uwot/index.html) package.
- The [**libscran**](https://github.com/LTLA/libscran) library implements high-level methods for scRNA-seq data analysis, ranging from quality control to clustering.
The code here originates from an assortment of Bioconductor packages - 
namely, [**scran**](https://bioconductor.org/packages/scran), [**scuttle**](https://bioconductor.org/packages/scuttle) and [**scater**](https://bioconductor.org/packages/scater) -
that we just bundled together into a single C++ library for convenience. 

We compile our C++ code to Wasm using the [Emscripten toolchain](https://emscripten.org/) to create Javascript-visible bindings.
This allows our Javascript code to easily call C++ functions and interact with instances of C++ classes via Wasm.
Compilation is largely painless though some adjustment is required to deal with the (mostly security-related) constraints of the browser.
In particular, code involving file input required some refactoring as the browser does not allow applications to directly access the file system.
Some additional gymnastics were required to enable multi-threading support with shared memory _(a single-threaded version of the application is also available)._

To pass input arrays from Javascript to Wasm, we allocate a buffer on the Wasm heap with Javascript, bind that buffer to a `TypedArray` view and fill it with the input values.
The offset is then passed as an integer to the Wasm binary, which is cast to a pointer of the relevant type to access the data at that location.
To pass results to Javascript, we typically return a persistent instance of a C++ class with methods to return `TypedArray` views of a contiguous array. 
This is convenient as it does not require the Javascript code to know the size and number of the output arrays ahead of time.
In both cases, the Javascript code must free the relevant memory after use to avoid a memory leak.
