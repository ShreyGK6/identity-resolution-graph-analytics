# DBLP–ACM Identity Resolution Using Snowflake and Graph Analytics

## Project Overview

This project implements an entity resolution workflow to identify matching academic research publications across two bibliographic datasets: DBLP and ACM.

Using Snowflake, Python, and graph analytics, the project processes publication records, normalizes research paper titles, generates candidate matches, and calculates similarity scores to identify potential duplicate publications.

The resulting matches are represented as a graph, where publication records are represented as nodes and matching relationships are represented as edges. Connected components are then used to identify potential groups of duplicate records.

The project also evaluates matching performance against a ground-truth mapping dataset using precision, recall, and F1-score.

## Objectives

- Load and manage benchmark datasets using Snowflake internal stages.
- Process DBLP and ACM publication records using Python and Snowpark.
- Normalize publication titles for consistent comparison.
- Generate candidate record pairs using token-based blocking.
- Apply fuzzy string matching to calculate title similarity.
- Construct a graph representing potential publication matches.
- Identify connected components and assign cluster IDs.
- Evaluate matching performance against ground-truth records.

## Technologies Used

| Category | Technologies |
|---|---|
| Cloud Data Platform | Snowflake |
| Data Processing | Python, Pandas, Snowpark |
| Graph Analytics | NetworkX |
| Fuzzy Matching | RapidFuzz |
| Data Analysis | Precision, Recall, F1-score |
| Development Environment | Snowflake Python Notebooks |
| Version Control | Git, GitHub |

## Dataset

This project uses the DBLP–ACM bibliographic record linkage benchmark.

The ZIP archive containing the three CSV files is uploaded to a Snowflake internal stage named `FACEBOOK_STAGE`.

The three files provide:
- DBLP publication records.
- ACM publication records.
- Ground-truth mappings used to evaluate matching performance.

The dataset is accessed through Snowflake rather than committed to this repository.

## Project Architecture

1. **Data Ingestion:** Upload the ZIP archive to a Snowflake internal stage.
2. **Data Access:** Retrieve the staged archive from the Snowflake Python Notebook runtime.
3. **Data Preparation:** Extract the CSV files within the notebook runtime and load the publication records into Pandas DataFrames.
4. **Normalization:** Standardize publication titles and assign unique node identifiers.
5. **Candidate Generation:** Use token-based blocking to reduce unnecessary record comparisons.
6. **Similarity Scoring:** Apply RapidFuzz to calculate title similarity scores.
7. **Graph Construction:** Represent publication records as nodes and qualifying matches as weighted edges.
8. **Clustering:** Identify connected components and assign cluster identifiers.
9. **Evaluation:** Compare predicted matches against ground-truth mappings using precision, recall, and F1-score.
10. **Snowflake Storage:** Prepare analytical outputs for storage in Snowflake tables.

## Matching Methodology

### 1. Title Normalization

Publication titles are converted to lowercase, punctuation is removed, and whitespace is standardized.

### 2. Candidate Generation

Token-based blocking identifies records that share informative title words. This reduces the number of record pairs requiring similarity calculations.

### 3. Fuzzy Matching

RapidFuzz calculates similarity scores between candidate publication titles.

The initial matching threshold is set to 90/100. This threshold is configurable and should be validated against the ground-truth dataset.

### 4. Graph Construction

NetworkX constructs an undirected graph:

- Nodes represent individual DBLP and ACM publication records.
- Edges represent candidate matches that pass the similarity threshold.
- Edge weights represent title similarity scores.
- Connected components identify groups of potentially matching records.

### 5. Model Evaluation

The predicted matches are compared against the ground-truth mapping dataset.

Evaluation metrics include:

- **Precision:** Proportion of predicted matches that are correct.
- **Recall:** Proportion of actual matches successfully identified.
- **F1-score:** Harmonic mean of precision and recall.

Both edge-level and cluster-level matching performance are evaluated.

## Repository Structure

```text
dblp-acm-identity-resolution/
│
├── README.md
├── notebooks/
│   └── DBLP_ACM_Identity_Resolution.ipynb
├── reports/
│   └── DBLP_ACM_Identity_Resolution_Report.html
├── sql/
│   └── snowflake_setup.sql
├── docs/
│   └── methodology.md
├── images/
├── requirements.txt
├── .gitignore
└── LICENSE
```

## Setup and Execution

### Prerequisites

- Snowflake account with access to Python Notebooks.
- An appropriate Snowflake database, schema, and internal stage.
- The DBLP–ACM benchmark ZIP archive.

### Python Dependencies

```bash
pip install pandas networkx rapidfuzz
```

Snowpark is provided through the Snowflake notebook environment.

### Snowflake Configuration

Create or configure your Snowflake database, schema, and internal stage.

The notebook currently expects:

```sql
USE DATABASE GRAPH_ANALYTICS_DB;
USE SCHEMA FACEBOOK;

LIST @FACEBOOK_STAGE;
```

Upload the benchmark ZIP archive to the internal stage before executing the notebook.

### Running the Project

1. Open the notebook in Snowflake.
2. Confirm the active Snowpark session.
3. Verify the ZIP archive is available in the internal stage.
4. Execute the notebook cells in order.
5. Review candidate matches, graph clusters, and evaluation metrics.
6. Optionally save the resulting analytical tables in Snowflake.

## Limitations and Future Improvements

- Incorporate author names, publication years, and venues into similarity scoring.
- Experiment with additional candidate-generation techniques.
- Optimize matching thresholds using validation data.
- Investigate one-to-one matching constraints to reduce incorrect transitive clustering.
- Add graph visualizations and interactive analytical dashboards.
- Explore scalable graph processing approaches for larger datasets.

## Disclaimer

This project is an educational entity resolution prototype. Similarity scores and connected components represent proposed record matches, not guaranteed identity matches.

The matching threshold and methodology should be validated before applying this approach to production data.

