
<!-- README.md is generated from README.Rmd. Please edit that file -->

# Robust Cell Type Decomposition (RCTD)

<!-- badges: start -->

<!-- badges: end -->

Welcome to RCTD, an R package for assigning cell types to spatial
transcriptomics data. RCTD inputs a spatial transcriptomics dataset,
which consists of a set of *pixels*, which are spatial locations that
measure RNA counts across many genes. RCTD additionally uses a single
cell RNA-seq (scRNA-seq) dataset, which is labeled for cell types. RCTD
learns cell type profiles from the scRNA-seq dataset, and uses these to
label the spatial transcriptomics pixels as cell types. Notably, RCTD
allows for individual pixels to be cell type *mixtures*; that is, they
can potentially source RNA from multiple cell types. RCTD identifies the
cell types on each pixel, and estimates the proportion of each of these
cell types. Additionally, RCTD has a platform effect normalization step,
which normalizes the scRNA-seq cell type profiles to match the *platform
effects* of the spatial transcriptomics dataset. A platform effect is
the tendency of a sequencing technology to capture individual genes at
different rates.

Code for generating the figures of our paper, Robust decomposition of
cell type mixtures in spatial transcriptomics, is located
[here](https://github.com/dmcable/RCTD/tree/dev/AnalysisPaper).

## Installation

You can install the current version of RCTD from
[GitHub](https://github.com/dmcable/RCTD) with:

``` r
# install.packages("devtools")
devtools::install_github("dmcable/RCTD")
```

### Downloading RCTD Folder

We reccommend downloading the ‘RCTD\_base’ folder, and adapting it to be
your working directory for RCTD. ‘RCTD\_base’ can be downloaded as
follows:

``` bash
#!/bin/bash
wget https://raw.githubusercontent.com/dmcable/RCTD/master/RCTD_base.zip 
unzip RCTD_base.zip
```

### Details of RCTD Folder Contents

For the rest of this document, we assume that you are working in
‘RCTD\_base’ as a base directory. This folder contains several
important subfolders/files:

  - conf: Configuration files required for running RCTD.
      - This folder must be placed in the directory where you are
        running RCTD.
      - Data locations are entered into the ‘conf/dataset.yml’ file, and
        RCTD parameters in ‘conf/default.yml’.
      - For selecting a different RCTD parameter configuration file, you
        can change the `config_mode` field in ‘conf/dataset.yml’ to
        point to e.g. ‘conf/default.yml’, ‘conf/test.yml’, or another
        file.
      - Set `n_puck_folds` for number of folds to split the dataset
        (RCTD runs on batches/folds of the dataset).
  - data: A recommended folder to contain the data. Contains two
    subfolders ‘Reference’ for containing the scRNA-seq data, and
    ‘SpatialRNA’ for containing the spatial transcriptomics data.
    Examples are included with the ‘Vignette’ subfolders. RCTD will
    place its results in the ‘SpatialRNA’ subdirectory.
  - R\_scripts: R scripts for running RCTD
      - See the following ‘Workflow’ and ‘Recommended Guidelines’
        sections for explanation of how to use these scripts.
  - bash\_scripts: bash scripts for submitting RCTD jobs to the computer
    cluster.
      - These sample scripts were configured for the Broad Institute’s
        PBS cluster, but they are not likely to directly work without
        modification on other clusters.
      - See the section ‘Running RCTD on a Cluster’ for more details.
  - ‘spatial-transcriptomics.Rmd’ a vignette for RCTD, which you can run
    after downloading the ‘RCTD\_base’ folder
      - This vignette explains most of the features of RCTD.

## Quick Guide to Getting Started with RCTD

Here, we assume that you have installed the RCTD R package and
downloaded the ‘RCTD\_base’ folder as above. In this section, we aim to
explain how to use RCTD as quickly as possible on your data:

1.  Within the ‘RCTD\_base’ folder, open the
    ‘spatial-transcriptomics.Rmd’ vignette in ‘RStudio.’ Run it for a
    complete explanation of the RCTD workflow.
2.  As described in the ‘Data Preprocessing’ step of the vignette, place
    your raw data in the ‘RCTD\_base/data/Reference/YOUR\_DATA’ and
    ‘RCTD\_base/data/SpatialRNA/YOUR\_DATA’ folders, where
    ‘YOUR\_DATA’ is the name of your current dataset. Next, also
    described in the vignette, convert and save this data as ‘RDS’
    objects.
3.  Set configuration files within the ‘RCTD\_base/conf’ folder. As
    described in the ‘Setup’ step of the vignette, you need to edit the
    `dataset.yml` file to point to your dataset. You should be able to
    build off of the `dataset_sample.yml` as follows:

<!-- end list -->

``` bash
#!/bin/bash
cd RCTD_base/conf
cp dataset_sample.yml my_dataset.yml # make a copy for your config file
# edit the my_dataset.yml config file to have your dataset file locations
cp my_dataset.yml dataset.yml # set your config file to be active
# now, when you run RCTD, it will run on your dataset
```

You should make sure that the `SpatialRNAfolder`,`puckrds`,`reffolder`,
and `reffile` fields are updated for your dataset. If you would like,
you can leave the `puckrds` and `reffile` fields fixed and just change
the folders. You can optionally set `config_mode` to ‘test’ to quickly
test RCTD, but you should set it to ‘default’ for the official run. For
Slide-seq datasets (for example), we reccommend setting `n_puck_folds
= 20,` which will split your `SpatialRNA` dataset into 20 batches.

4.  Run RCTD
      - Option 1: Run RCTD on the cluster. If you have access to Broad
        Institute’s PBS cluster, run the
        ‘RCTD\_base/bash\_scripts/pipeline\_sample.sh’ (you must
        change the ‘YOUR\_PATH’ fields to your ‘RCTD\_base’ location).
        Otherwise, you may modify this script for working on your
        particular cluster. For more details on how to do so, see the
        ‘Running RCTD from Command Line’ and ‘Running on a Cluster’
        sections below.
      - Option 2: Run RCTD within a single R session. Set `n_puck_folds
        = 1` within your ‘RCTD\_base/dataset.yml’ file, and follow the
        ‘spatial-transcriptomics.Rmd’ vignette.
      - Option 3: Run RCTD from command line. See the ‘Running RCTD from
        Command Line’ section below.
5.  Process RCTD results. Obtain results objects and make summary plots.
      - Run the ‘RCTD\_base/R\_scripts/gather\_results.R’ script or,
        equivalently, follow the ‘Collecting RCTD results’ section of
        the ‘spatial-transcriptomics’ vignette.

## Detailed Guide to RCTD

The basic workflow for RCTD consists of the following steps:

1.  Data Preprocessing. Representing the spatial transcriptomics data as
    a `SpatialRNA` object, and the scRNA-seq reference as a `Seurat`
    object.
      - Save these objects as ‘RDS’ files, as shown in the ‘Data
        Preprocessing’ section of the ‘spatial-transcriptomics’
        vignette.
2.  Setup. Edit the configuration files (e.g. ‘conf/dataset.yml’ and
    ‘conf/default.yml’) to point to the data files and e.g. determine
    parameters for selecting differentially expressed genes.
      - Follow the ‘Setup’ section of the ‘spatial-transcriptomics’
        vignette.
3.  Platform Effect Normalization. Estimates platform effects between
    scRNA-seq dataset and spatial transcriptomics dataset. Uses this to
    normalize cell type profiles.
      - Run the ‘R\_scripts/fitBulk.R’ script or, equivalently, follow
        the ‘Platform Effect Normalization’ section of the
        ‘spatial-transcriptomics’ vignette.
4.  Hyperparameter optimization (choosing sigma). Handles overdispersion
    by determining the maximum likelihood variance for RCTD’s lognormal
    random effects. Precomputes likelihood function.
      - Run the ‘R\_scripts/chooseSigma.R’ script or, equivalently,
        follow the ‘Hyperparameter optimization (choosing sigma)’
        section of the ‘spatial-transcriptomics’ vignette.
5.  Robust Cell Type Decomposition. Assigns cell types to each spatial
    transcriptomics pixel, and estimates the cell type proportions.
      - Run the ‘R\_scripts/callDoublets.R’ for each data fold. An
        example of how this works is outlined in the ‘Robust Cell Type
        Decomposition’ section of the ‘spatial-transcriptomics’
        vignette.
6.  Collecting RCTD results. Obtain results objects and make summary
    plots.
      - Run the ‘R\_scripts/gather\_results.R’ script or, equivalently,
        follow the ‘Collecting RCTD results’ section of the
        ‘spatial-transcriptomics’ vignette.

### Running RCTD from Command Line

We recommend running steps 1-2 in the R console (e.g. in RStudio), as
demonstrated in the ‘spatial-transcriptomics’ vignette. After setting
the configuration files, we recommend that steps 3-5 are run from
command line as follows (where there are, for example, three data
folds):

``` bash
#!/bin/bash
Rscript R_scripts/fitBulk.R
Rscript R_scripts/chooseSigma.R
Rscript R_scripts/callDoublets.R 1 # first fold of data
Rscript R_scripts/callDoublets.R 2 # second fold of data
Rscript R_scripts/callDoublets.R 3 # third fold of data
Rscript R_scripts/gather_results.R
```

Note that, for Slide-seq, we typically reccommend setting `n_puck_folds
= 20,` in which case the ‘callDoublets.R’ script would need to be run 20
separate times. Step 6 (gathering RCTD results) can be optionally run
from command line as above or in the R console.

### Running RCTD on a Cluster

One (recommended) option is to automate this workflow on a computer
cluster for steps 3-5. Each of steps 3-5 must be run sequentially, but
each fold may be run in parallel (as a job array) for step 5. We provide
an example of scripts used to automate RCTD on the Broad Institute’s PBS
cluster in the ‘bash\_scripts’ folder. Note that other clusters may have
different syntax for submitting jobs. If ‘pipeline\_sample.sh’ is edited
to have the correct number of folds, and ‘pipeline\_sample.sh’,
‘doub\_sample.sh’, and ‘pre\_sample.sh’ are edited with paths to the
RCTD folder, then one can, for example, run RCTD by the following
command:

``` bash
#!/bin/bash
bash ./bash_scripts/pipeline_sample.sh
```

### Dependencies

  - R version \>= 3.5.
  - R packages: caret, readr, config, Seurat, pals, ggplot2, Matrix,
    doParallel, foreach, quadprog, tibble, dplyr, reshape2.

For optimal performance, we recommend at least 4 GB of RAM, and multiple
cores may be used to speed up runtime.

Installation time: Less than two minutes, after installing dependent
packages.

Runtime: The example dataset provided (Vignette) can be run in less than
10 minutes on a normal desktop computer. Approximately 16 hours (on
Broad Institute computer cluster) for each Slide-seq dataset (tested on
cerebellum and hippocampus datasets with 10,000 - 25,000 pixels).

Operating systems (version 1 RCTD) tested on:

  - macOS Mojave 10.14.6
  - GNU/Linux (GNU coreutils) 8.22

### License

RCTD is [licensed](https://github.com/dmcable/RCTD/blob/master/LICENSE)
under the GNU General Public License v3.0.
