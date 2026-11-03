# DBLP--ACM Identity Resolution Methodology

## 1. Project Overview

This project implements an entity resolution workflow to identify
potentially duplicate academic publication records across the DBLP and
ACM bibliographic datasets.

The workflow uses publication-title normalization, token-based candidate
generation, fuzzy string similarity, and graph-based clustering.
Matching results are evaluated against a ground-truth mapping file
supplied with the benchmark dataset.

The implementation is developed in a Snowflake Python Notebook using
Python, Snowpark, Pandas, RapidFuzz, and NetworkX.

## 2. Dataset Description

The notebook uses three CSV files from the DBLP--ACM benchmark archive:

  -----------------------------------------------------------------------
  File                                Purpose
  ----------------------------------- -----------------------------------
  `DBLP2.csv`                         DBLP publication records

  `ACM.csv`                           ACM publication records

  `DBLP-ACM_perfectMapping.csv`       Ground-truth DBLP--ACM record pairs
                                      used for evaluation
  -----------------------------------------------------------------------

The source publication files contain the following fields:

-   `id`: Source record identifier
-   `title`: Publication title
-   `authors`: Publication authors
-   `venue`: Publication venue
-   `year`: Publication year

The notebook loads the DBLP and ACM source records for matching. The
perfect-mapping file is used to evaluate predicted matches rather than
to generate candidate pairs.

The saved notebook output reports 2,616 DBLP records and 2,294 ACM
records.

## 3. Snowflake Data Access and Ingestion

The dataset archive is uploaded to a Snowflake internal stage named
`@FACEBOOK_STAGE`.

The notebook:

1.  Retrieves the active Snowpark session with `get_active_session()`.
2.  Selects the `GRAPH_ANALYTICS_DB` database and `FACEBOOK` schema.
3.  Lists the files available in `@FACEBOOK_STAGE`.
4.  Downloads the staged ZIP archive into a temporary directory in the
    notebook runtime using `session.file.get()`.
5.  Extracts the archive with Python's `zipfile` module.
6.  Locates the CSV files for subsequent processing.

The notebook also supports staged CSV files when no ZIP archive is
present.

**Execution detail:** The CSV files are not queried directly as tables
while compressed in the stage. The current implementation downloads and
extracts the archive in the notebook's temporary runtime, then loads the
extracted files with Pandas. The original ZIP and CSV files do not need
to be committed to this GitHub repository.

## 4. Data Preparation and Record Identification

The notebook identifies the DBLP and ACM source files by their filenames
and excludes files whose names indicate that they contain mapping data.

The two source CSV files are loaded into separate Pandas DataFrames. The
notebook prints their dimensions and column names and displays sample
records so the input structure can be checked.

Each source record is assigned a unique graph node identifier:

-   DBLP records use identifiers prefixed with `DBLP_`.
-   ACM records use identifiers prefixed with `ACM_`.

These identifiers distinguish records from the two datasets when they
are represented together in one graph.

## 5. Title Normalization

The current matching implementation uses publication titles.

Before comparison, each title is normalized by:

1.  Converting the value to lowercase.
2.  Replacing characters outside the letters, numbers, and spaces with
    spaces.
3.  Collapsing repeated whitespace.
4.  Trimming leading and trailing whitespace.

The normalized value is stored in a `NORMALIZED_TITLE` column. Missing
title values are converted to empty strings.

Although the source files also contain authors, venue, and year, those
attributes are not currently included in the similarity score. The
matching score in this implementation is based on title text only.

## 6. Candidate Pair Generation

Comparing every DBLP record with every ACM record would require
evaluating the full cross-dataset Cartesian product. The notebook
reduces the number of comparisons through token-based blocking.

The candidate-generation process is:

1.  Split each normalized ACM title into tokens.
2.  Exclude a predefined set of stopwords.
3.  Ignore tokens shorter than four characters.
4.  Build an index that maps each qualifying token to the ACM records
    containing it.
5.  For each DBLP title, retrieve ACM records that share at least one
    qualifying token.
6.  Store each distinct DBLP--ACM pair as a candidate pair.

This process narrows the set of pairs that proceed to similarity
scoring. It is a candidate-generation strategy, not a final matching
decision.

The saved notebook output reports **666,074 candidate pairs**.

## 7. Similarity Scoring and Match Threshold

The notebook uses the `ratio` function from `rapidfuzz.fuzz` to compare
the normalized title of each candidate DBLP record with the normalized
title of its candidate ACM record.

The resulting title similarity score is on a 0--100 scale. The initial
threshold is configured as:

``` python
MATCH_THRESHOLD = 90
```

A candidate pair is retained as a match candidate when its title
similarity score is greater than or equal to the configured threshold.

The resulting edge data includes:

  Field                  Description
  ---------------------- ---------------------------------------
  `SOURCE_ID`            DBLP node identifier
  `TARGET_ID`            ACM node identifier
  `TITLE_SIMILARITY`     Fuzzy title similarity score
  `IS_MATCH_CANDIDATE`   Whether the score meets the threshold

The saved notebook output reports **2,456 edges** at the threshold of
90.

The threshold is configurable. A score above the threshold indicates a
candidate relationship under this rule; it does not independently prove
that two records represent the same publication.

## 8. Graph Construction

NetworkX is used to build an undirected graph.

-   **Nodes** represent individual DBLP and ACM publication records.
-   **Edges** represent candidate DBLP--ACM relationships that meet the
    similarity threshold.
-   **Edge weights** store the title similarity score.

The notebook adds all DBLP and ACM records as nodes, then adds the
qualifying candidate relationships as weighted edges.

The saved notebook output reports the following graph statistics:

  Metric                           Result
  ------------------------------ --------
  Graph nodes                       4,910
  Graph edges                       2,456
  Connected components              2,684
  Nodes with at least one edge      4,322

These figures describe the saved notebook execution and may change if
the input data, matching threshold, or implementation changes.

## 9. Connected Components and Cluster Assignment

The notebook uses NetworkX connected components to group nodes that are
connected by one or more edges. Each connected component is assigned a
cluster identifier (`CLUSTER_ID`).

Isolated nodes are included as singleton clusters.

A connected component represents a proposed group of related records,
not a guarantee that every record in the group refers to the same
publication. Connected components are transitive: if record A is
connected to B and B is connected to C, all three records belong to the
same component even if A and C do not have a direct edge or do not
independently meet the similarity threshold.

This is an important limitation to consider when interpreting
cluster-level results.

## 10. Ground-Truth Evaluation

The `DBLP-ACM_perfectMapping.csv` file supplies known DBLP--ACM record
pairs for evaluation. It is kept separate from the candidate-generation
and title-scoring steps.

The notebook evaluates predictions at two levels:

### Edge-level evaluation

This evaluation compares the individual DBLP--ACM edges retained by the
matching threshold with the known pairs in the ground-truth mapping.

### Cluster-level evaluation

This evaluation considers cross-dataset DBLP--ACM pairs grouped within
the same connected component. It can include pairs connected through a
path of multiple edges, even when those pairs do not have a direct edge.

The notebook calculates:

-   **True positives (TP):** Predicted pairs that are present in the
    ground truth.
-   **False positives (FP):** Predicted pairs that are not present in
    the ground truth.
-   **False negatives (FN):** Ground-truth pairs that were not
    predicted.

It uses these counts to calculate:

-   **Precision:** The proportion of predicted pairs that are correct.
-   **Recall:** The proportion of ground-truth pairs that are
    identified.
-   **F1-score:** The harmonic mean of precision and recall.

The notebook also includes a threshold sweep from 80 through 100 in
increments of two to examine how changing the title-similarity threshold
affects edge-level matching performance.

Evaluation values should be taken from an actual execution of the
notebook. They are not stated here because the saved notebook output
does not provide the final metric values in the reviewed execution.

## 11. Optional Snowflake Output Tables

The notebook includes an optional section for writing processed results
back to Snowflake using `session.write_pandas()`.

The intended output tables are:

  -----------------------------------------------------------------------
  Table                               Intended contents
  ----------------------------------- -----------------------------------
  `DBLP_IDENTITY_NODES`               Processed DBLP publication records

  `ACM_IDENTITY_NODES`                Processed ACM publication records

  `DBLP_ACM_MATCH_EDGES`              Candidate matching edges and
                                      similarity scores

  `DBLP_ACM_GRAPH_CLUSTERS`           Graph cluster assignments
  -----------------------------------------------------------------------

Saving these tables is optional. The database and schema must be
configured for the user's Snowflake environment before the save
operation is run.

## 12. Limitations and Future Improvements

The current implementation is a title-based prototype. Its results
depend on title normalization, token-based blocking, the RapidFuzz
similarity function, and the selected threshold.

Potential improvements include:

-   Incorporating author, venue, and publication-year information into a
    multi-attribute matching approach.
-   Testing additional blocking strategies to improve candidate coverage
    and reduce unnecessary comparisons.
-   Comparing alternative string similarity methods for titles with
    substantial formatting or wording differences.
-   Selecting thresholds using a validation process and reviewing
    precision--recall trade-offs.
-   Reviewing false-positive and false-negative examples.
-   Inspecting large or unexpected connected components for transitive
    false matches.
-   Adding a manual review step for ambiguous candidate pairs.
-   Building a Power BI report from saved Snowflake outputs to summarize
    matching and graph results.

## 13. Reproducibility and Execution Environment

To reproduce the workflow, a user needs:

-   Access to a Snowflake account and permission to use a Snowflake
    Python Notebook.
-   An active Snowpark session in the notebook.
-   A database and schema that the user can access.
-   The DBLP--ACM benchmark ZIP archive or its CSV files uploaded to a
    Snowflake stage.
-   The Python packages listed in `requirements.txt`.

The notebook currently uses the following database and schema names:

``` python
DATABASE_NAME = "GRAPH_ANALYTICS_DB"
SCHEMA_NAME = "FACEBOOK"
```

It accesses the stage as `@FACEBOOK_STAGE`. Users should update these
settings and the stage reference if their Snowflake environment uses
different names.

The notebook uses the active Snowpark session provided by Snowflake. The
required Python packages should be available in the notebook
environment; see `requirements.txt` for the project's package list.

## 14. Conclusion

This project demonstrates an entity resolution workflow that combines
Snowflake-based file access, Python data preparation, token-based
candidate generation, fuzzy title matching, graph construction,
connected-component clustering, and evaluation against known record
pairs.

The implementation provides a reproducible starting point for
identifying potentially duplicate academic publication records across
DBLP and ACM. Its results should be interpreted as proposed matches and
clusters that require evaluation, rather than definitive identity
decisions.
