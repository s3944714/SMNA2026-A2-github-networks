# COSC 2671 — Social Media and Network Analysis
## Assignment 2 — Group Project

---

## Project Title
**Who Drives Open-Source AI? Network Centrality and Discourse Analysis on GitHub ML Repositories**

---

## Team

| Name | Student ID | Role |
|---|---|---|
| John Robyn De Guzman | s3944714 | Data collection, network analysis, NLP analysis, report writing |
| Phavan Kuumar Janakiraman Suresh | s4190557 | Literature review, background and methodology writing, discussion |

**Group:** 17  


---

## Research Question

Do structurally central contributors in top open-source AI/ML GitHub repositories produce qualitatively different issue discourse than peripheral contributors — and can distinct collaboration communities be identified through network structure and topic modelling?

---

## Repositories Analysed

| Repository | Domain |
|---|---|
| scikit-learn/scikit-learn | Classical machine learning |
| huggingface/transformers | Large-scale NLP and transformer models |
| pytorch/pytorch | Deep learning framework |
| keras-team/keras | High-level neural network API |
| langchain-ai/langchain | LLM application framework |

---

## Requirements

### Python Version
Python 3.10 or higher is recommended.  
Environment: `conda base` (Python [conda env:base])

### Install All Dependencies
Run **Notebook 1, Cell 2** to install all packages automatically, or install manually:

```bash
pip install PyGithub python-dotenv pandas numpy networkx \
            python-louvain bertopic sentence-transformers \
            vaderSentiment pyvis matplotlib seaborn \
            plotly scipy tqdm scikit-learn
```

---

## GitHub Token Setup (Required Before Running Notebook 1)

Notebook 1 requires a GitHub Personal Access Token to authenticate with the GitHub REST API.

**Steps:**
1. Go to GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)
2. Click **Generate new token**
3. Select scope: `public_repo`
4. Copy the generated token
5. Create a file named `.env` in the project root directory:

```
GITHUB_TOKEN=your_token_here
```

---

## Project Directory Structure

```
SMNA2026-A2-github-networks/
│
├── .env                                        ← GitHub token (NOT submitted, gitignored)
├── README.md                                   ← This file
├── Access_s3944714_PG_Group_17.txt             ← Repository access details
│
├── data/
│   ├── raw/                                    ← Created by Notebook 1
│   │   ├── huggingface_transformers_issues.json
│   │   ├── keras-team_keras_issues.json
│   │   ├── langchain-ai_langchain_issues.json
│   │   ├── pytorch_pytorch_issues.json
│   │   └── scikit-learn_scikit-learn_issues.json
│   │
│   ├── processed/                              ← Created by Notebook 2 (gitignored)
│   │   ├── *_clean.csv                         ← Cleaned issue/comment data
│   │   ├── *_sentiment.csv                     ← VADER-scored data (Notebook 4)
│   │   ├── *_network.gexf                      ← Network files (Notebook 3)
│   │   └── all_centrality.pkl                  ← Centrality data passed to Notebook 4
│   │
│   └── sample/                                 ← 10-row samples (submitted)
│       ├── sample_scikit-learn_scikit-learn_clean.csv
│       ├── sample_huggingface_transformers_clean.csv
│       ├── sample_pytorch_pytorch_clean.csv
│       ├── sample_keras-team_keras_clean.csv
│       └── sample_langchain-ai_langchain_clean.csv
│
├── notebooks/
│   ├── s3944714_PG_Group_17_01_data_collection.ipynb
│   ├── s3944714_PG_Group_17_02_cleaning.ipynb
│   ├── s3944714_PG_Group_17_03_network_analysis.ipynb
│   └── s3944714_PG_Group_17_04_nlp_analysis.ipynb
│
├── outputs/
│   ├── figures/                                ← All PNG charts and HTML networks
│   │   ├── network_summary_table.png
│   │   ├── community_size_distribution.png
│   │   ├── network_comparison.png
│   │   ├── *_top20_centrality.png
│   │   ├── sentiment_distribution.png
│   │   ├── sentiment_temporal_trend.png
│   │   ├── topic_sizes.png
│   │   ├── centrality_vs_sentiment.png
│   │   └── networks/
│   │       └── *_network.html                  ← Interactive pyvis network visualisations
│   │
│   └── tables/                                 ← All CSV result tables
│       ├── network_summary.csv
│       ├── mannwhitney_results.csv
│       ├── spearman_results.csv
│       ├── nlp_summary.csv
│       ├── *_centrality.csv
│       ├── *_centrality_full.csv
│       ├── *_centrality_sentiment.csv
│       └── *_topic_info.csv
```

---

## Run Order — Must Be Followed in Sequence

> **All notebooks must be run in order 01 → 02 → 03 → 04.**  
> Each notebook depends on outputs from the previous one.

---

### Notebook 1 — Data Collection
**File:** `s3944714_PG_Group_17_01_data_collection.ipynb`

| | |
|---|---|
| **Purpose** | Collect issues and comments from GitHub REST API via PyGitHub 2.x |
| **Input** | `.env` file containing `GITHUB_TOKEN` |
| **Output** | `data/raw/{repo}_issues.json` — 5 JSON files, one per repository |
| **Runtime** | 4–8 hours at `MAX_ISSUES=2000` — run overnight |

**Before running:**
- Ensure `data/raw/` directory exists. Create manually if needed: `mkdir -p data/raw`
- Ensure `.env` file is present with a valid GitHub token

**Quick test:**
- In Cell 4 (Config), reduce `MAX_ISSUES = 100` to test without the full overnight run
- Restore to `MAX_ISSUES = 2000` for the full collection

---

### Notebook 2 — Data Cleaning
**File:** `s3944714_PG_Group_17_02_cleaning.ipynb`

| | |
|---|---|
| **Purpose** | Flatten raw JSON → clean CSVs; remove bots; redact credentials |
| **Input** | `data/raw/{repo}_issues.json` (from Notebook 1) |
| **Output** | `data/processed/{repo}_clean.csv` — 5 cleaned CSV files |
| **Runtime** | ~3–5 minutes |

**Notes:**
- Removes 23 bot accounts including pytorch-bot[bot] (1,527 posts) and google-ml-butler[bot] (1,055 posts)
- Redacts exposed API credentials using regex patterns (AWS keys, OpenAI keys)
- Processed CSVs are gitignored due to credential redaction — regenerate by re-running this notebook
- `clean_text` field strips Markdown, code blocks, URLs, HTML tags, and @-mentions

---

### Notebook 3 — Network Analysis
**File:** `s3944714_PG_Group_17_03_network_analysis.ipynb`

| | |
|---|---|
| **Purpose** | Build directed interaction networks; compute centrality; detect communities; k-core decomposition |
| **Input** | `data/processed/{repo}_clean.csv` (from Notebook 2) |
| **Output** | `all_centrality.pkl`, GEXF files, centrality CSVs, figures, pyvis HTML networks |
| **Runtime** | ~10–15 minutes |

**Network design:**
- **Nodes:** GitHub contributors (authors of issues and comments)
- **Edges:** Directed — A → B when user A comments on an issue opened by B
- **Weight:** Number of such interactions between A and B
- **Filter:** Nodes with degree ≥ 2; largest weakly connected component only

**Centrality measures:**
- PageRank (α = 0.85, weighted)
- Normalised weighted betweenness centrality
- Composite score = 0.4 × PageRank + 0.3 × Betweenness + 0.3 × Norm In-Degree

**Community analysis:**
- Louvain community detection on undirected projection (random seed = 42)
- K-core decomposition — maximum core and overlap with top-30 PageRank nodes

**Important:** This notebook saves `all_centrality.pkl` which is required by Notebook 4.

---

### Notebook 4 — NLP Analysis
**File:** `s3944714_PG_Group_17_04_nlp_analysis.ipynb`

| | |
|---|---|
| **Purpose** | VADER sentiment scoring; Mann-Whitney U test; BERTopic topic modelling; Spearman correlation |
| **Input** | `data/processed/{repo}_clean.csv` (from Notebook 2) + `all_centrality.pkl` (from Notebook 3) |
| **Output** | Sentiment CSVs, topic info CSVs, result tables, figures |
| **Runtime** | ~20–30 minutes (BERTopic embedding is the slowest step) |

**Analysis steps:**
- VADER compound scores computed for all 42,550 rows
- Mann-Whitney U test (two-sided): open vs closed issue sentiment (SC5)
- BERTopic with `all-MiniLM-L6-v2` embeddings, bigram CountVectorizer (min_df=3, max_df=0.90, min_topic_size=15)
- Spearman rank correlation: user PageRank vs mean VADER sentiment (users with ≥3 posts only)

>  `all_centrality.pkl` **must exist** before running this notebook.  
> Run Notebook 3 completely before starting Notebook 4.

---

## Success Criteria

| Criterion | Requirement | Result |
|---|---|---|
| SC1 | ≥100 nodes and ≥200 edges per repo |  Passed — all 5 repos |
| SC2 | Top-20 central contributors identified per repo | Passed — all 5 repos |
| SC3 | Louvain modularity > 0.30 per repo |  Passed — range 0.38–0.65 |
| SC4 | BERTopic ≥5 coherent topics per repo |  Passed 4/5 — pytorch returned 3 topics |
| SC5 | Open vs closed issue sentiment p < 0.05 |  Passed — all 5 repos (p ≤ 0.006) |
| SC6 | Spearman centrality-sentiment significant |  Partial — significant in pytorch and langchain |
| SC7 | Cross-repository comparison table | Passed — see outputs/tables/network_summary.csv |

---

## Data Sample Note

Full processed CSVs are not submitted due to file size and credential redaction. A 10-row representative sample from each repository is included in `data/sample/` to demonstrate the data structure and column schema.

To regenerate the full dataset, follow the run order above with a valid GitHub token.

**Columns in cleaned CSVs:**
`id, type, author, text, clean_text, state, created_at, closed_at, parent_id, parent_author, labels, issue_number, comment_count`

---

## Key Findings

1. All five repository networks exhibit meaningful community structure (Louvain modularity 0.38–0.65, SC3 passed)
2. Open issues carry significantly more positive sentiment than closed issues across all repositories (SC5 passed, all p ≤ 0.006)
3. A consistent negative Spearman correlation between PageRank and mean sentiment is observed across all five repositories — most central contributors post with lower positivity, interpreted as a signature of maintainer governance behaviour rather than community dysfunction
4. Correlation reaches statistical significance in pytorch/pytorch (ρ=−0.12, p=0.045) and langchain-ai/langchain (ρ=−0.22, p<0.001)

---

## Academic Integrity

All code is original. Standard Python packages (NetworkX, BERTopic, VADER, etc.) are used as libraries only. No existing end-to-end solution has been copied. All data was collected from public GitHub repositories via the official REST API in compliance with GitHub's terms of service.

---

## Contact

| Name | Email |
|---|---|
| John Robyn De Guzman | s3944714@student.rmit.edu.au |
| Phavan Kuumar Janakiraman Suresh | s4190557@student.rmit.edu.au |
