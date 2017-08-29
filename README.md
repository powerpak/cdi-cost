# cdi-cost

This code implements an analysis of changes in length of stay associated with *Clostridium difficile* infection using machine learning on electronic medical record (EMR) data, reported in Pak et al. 2017 (in press at [*Infect Control Hosp Epidemiol*][iche]). The code shows the full process of fitting propensity models using [elastic net regularized](https://en.wikipedia.org/wiki/Elastic_net_regularization) logistic regression, [propensity score matching](https://en.wikipedia.org/wiki/Propensity_score_matching), and subsequent statistical comparisons. The process starts from the data exported from our EMR, which is in the tab-separated values (TSV) format illustrated by `data/exported_visit_data.EXAMPLE.tsv`.

Because the full 7 year, 171,938 row dataset was created in the course of routine clinical operations at [The Mount Sinai Hospital](http://www.mountsinai.org/), it is proprietary to Mount Sinai and may contain private patient information that remains identifiable despite our use of [Safe Harbor deidentification procedures](https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/#safeharborguidance); therefore we cannot share our input data publically. However, we provide these notebooks for full transparency on our statistical procedures and the creation of all figures in the article, and so that these methods may be re-used for any EMR data that is formatted in shape of our input files.

[iche]: https://www.cambridge.org/core/journals/infection-control-and-hospital-epidemiology

## Prequisites

You will need to install [R](https://www.r-project.org/) (we used version 3.2.2), [Jupyter](http://jupyter.org/), and the [R kernel for Jupyter](https://github.com/IRkernel/IRkernel).

The notebooks use many R packages, but these are all available on [CRAN](https://cran.r-project.org/). The packages and versions we used are as follows:

| library      | version |
| ------------ | ------- |
| `glmnet`     | 2.0-10  |
| `doMC`       | 1.3.4   |
| `ROCR`       | 1.0-7   |
| `gplots`     | 3.0.1   |
| `MatchIt`    | 2.4-21  |
| `Hmisc`      | 4.0-1   |
| `ggplot2`    | 2.2.1   |
| `cowplot`    | 0.6.3   |
| `simpleboot` | 1.1-3   |
| `UpSetR`     | 1.3.1   |
| `survival`   | 2.40-1  |
| `lattice`    | 0.20-34 |
| `parallel`   | 3.2.2   |
| `etm`        | 0.6-2   |

If while executing a notebook, you receive the error "there is no package called...", run

    install.packages("name-of-the-package")

within your R console to fix the problem.

## Viewing the code

The code is in 5 [Jupyter notebooks](http://jupyter-notebook-beginner-guide.readthedocs.io/en/latest/what_is_jupyter.html), which interleave the code we wrote with the outputs we generated and notes on our intentions and observations. Github allows you to view these notebooks by simply clicking on the `.ipynb` files above. They are numbered in the order of execution.

To view the notebooks locally on your own machine, install the above prerequisites, clone this repository, and then execute within this directory:

    jupyter nbconvert --to html *.ipynb

which will produce HTML versions of all of the notebooks.

## Running the code

Jupyter makes it easy to run our code on your own data. You will first want to edit and/or replace the files in `data/` with versions appropriate to your own dataset.

### Formatting your data

Note that all inputs for our analysis are TSV text files. Files with the `.tsv` extension contain a header row, while `.txt` indicates there is no header row and only one column of data. Crucially, we **do not** use any quoting or escaping in these files; this means that no field can contain a tab character. Fields can, however, contain unescaped quote `"` and apostrophe `'` characters. If this is not the case for your data, you may need to modify invocations of `read.table()` in the notebooks.

The `data/codelist_*` files are TSV files listing the codes for admission sources, lab results, medications, diagnoses, and surgery procedures that were observed in our EMR dataset. These are necessary to list separately from the data because some of them (medication codes and diagnosis codes) associate with human-readable descriptions that are useful to examine in later stages of the analysis, and having a complete list beforehand simplifies creation of the sparse matrix in the first notebook.

The `data/exported_visit_data.EXAMPLE.tsv` file contains a 4-row example of input data to the analysis (with field values extracted randomly from our dataset). You can copy this to `data/exported_visit_data.tsv` and use it as a template for your own EMR visit data. Each row represents data from one patient visit. The fields (columns) are as follows:

- `length_of_stay` – number of days from admission to discharge
- `cdi_dx` – whether the visit was assigned an ICD-9 diagnosis code of 008.45, as `Y` or `N`
- `cdtox_pcr_positive` – whether the visit had a positive PCR toxin assay, as `Y` or `N`
- `cdtox_eia_positive` – whether the visit had a positive EIA toxin assay, as `Y` or `N`
- `started_cdtox_positive` – whether the PCR toxin assay was the standard lab test during the visit, as `Y` or `N`
- `cdtox_positive_after` – the number of days into the visit until the first positive toxin assay
- `age` – in years, as an integer. Note that values of 90 reflect "90 and up"
- `age_over_89`– if the patient's age is over 89 (i.e. "90 and up"), as `Y` or `N`
- `gender` – either `Male`, `Female`, `Indeterminant`, or `NOT AVAILABLE`
- `admission_sources` – any number of values from `codelist_adm_sources.txt` separated by pipe `|` characters
- `surgery_cases` – any number of `code` values from `codelist_surgery_procs.txt` separated by pipe `|` characters
- `problem_list` – any number of `code` values from `codelist_seen_dxs.txt` separated by pipe `|` characters
- `meds_reported` – any number of `code` values from `codelist_med_codes.txt` separated by pipe `|` characters
- `meds_administered` – any number of `code` values from `codelist_med_codes.txt` separated by pipe `|` characters
- `abnormal_labs` – any number of `code` values from `codelist_lab_result_codes.txt` separated by pipe `|` characters

Note that in our data, the last give columns only reflect values for the first 24 hours of each admission, in order to not unfairly include knowledge from after a CDI diagnosis during propensity modeling.

### Running the analysis

Once your data is in place, ensure all prerequisites above are installed, and run

    jupyter notebook

from this directory. Your web browser should open and allow you to view and run code in each of the notebooks.