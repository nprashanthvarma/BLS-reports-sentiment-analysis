# Reading the Labor Market: Text Analytics of BLS Employment Reports

> **Does the language of the U.S. Bureau of Labor Statistics' monthly jobs report reflect real labor-market conditions?**
> An NLP project over **121 BLS *Employment Situation* releases (Jan 2016 – Feb 2026)**, compared with the FRED unemployment rate.

*DSC 785 Text Analytics, Adelphi University, Spring 2026. Final group project (team of four).*
**My contribution**

- Designed the end-to-end project architecture, from BLS PDFs to final findings
- Built the text-extraction and cleaning pipeline (121 reports, stop-words, lemmatisation)
- Built the TF-IDF and LDA topic-modelling analysis

![Workflow](workflow_simple.png)

## At a glance

| | |
|---|---|
| **Data** | 121 monthly BLS PDF reports (about 43 pages each) and the FRED `UNRATE` monthly series |
| **Methods** | Text extraction, cleaning and lemmatising, TextBlob sentiment, LDA topic modelling, TF-IDF, logistic regression, Pearson/Spearman correlation |
| **Stack** | Python, pandas, pypdfium2, scikit-learn, gensim, TextBlob, SciPy, matplotlib |
| **Reproducible** | `python run_pipeline.py` rebuilds every chart and number in about 30 seconds; unit tests included |
| **Headline** | Report **topics** track the economic regime (COVID, annual revisions). Report **tone** barely tracks unemployment |

## What we found

1. **Topics tell the story; tone doesn't.** LDA separates a *pandemic / coronavirus / temporary layoff* regime (2020–2022) from routine reporting and from the *annual benchmark and seasonal-adjustment revision* language that appears every February.
2. **Sentiment is a weak signal.** Mean polarity is +0.03 (std 0.03), because the reports are written in deliberately neutral prose. Correlation with the unemployment rate is *r* = +0.21 overall, but that is driven by the COVID spike. Excluding Mar-2020 to Dec-2021 it is *r* = −0.22 (p = 0.03). There is no relationship with the month-to-month *change* (*r* = −0.04, p = 0.67).
3. **Text alone does not beat guessing at "did unemployment rise?"** With a time-ordered split and a majority-class baseline, accuracy is 56.7% for both the model and the baseline (30 test months). Macro-F1 is 0.57 vs 0.36. This is reported as a negative result rather than tuned until it looks good.

![Sentiment vs unemployment](sentiment_vs_unemployment.png)
![LDA topics over time](lda_topics_over_time.png)

<details><summary>More charts (word cloud, TF-IDF)</summary>

![Word cloud](wordcloud.png)
![TF-IDF](tfidf_top_terms.png)
</details>

## How it works

![System architecture](architecture.svg)

| Stage | Where | What it does |
|---|---|---|
| Extract | `src/bls_sentiment/extract.py` | Reads each PDF and takes the **reference month from the report title** (not the filename) |
| Clean | `preprocess.py` | Keeps only the narrative before the statistical tables, strips URLs and numbers, removes stop-words and boilerplate, lemmatises |
| Join | `analysis.merge_on_month` | Joins reports to FRED by month, so gaps in the series cannot shift the alignment |
| Analyse | `analysis.py` | Sentiment, correlations (including a COVID-excluded check), LDA (4 topics), TF-IDF, classifier vs baseline |
| Report | `plots.py`, `run_pipeline.py` | Figures to `results/figures/`, numbers to `results/metrics.json`, per-report scores to `results/report_scores.csv` |

## Run it

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python run_pipeline.py          # regenerates everything in results/
pytest -q                       # unit tests
jupyter lab notebooks/01_walkthrough.ipynb
```

No API keys are needed and nothing is downloaded at run time.

## Repository layout

```
run_pipeline.py        one-command entry point
src/bls_sentiment/     extract, preprocess, analysis, plots, config
tests/                 unit tests for date alignment, cleaning and join logic
notebooks/             01_walkthrough.ipynb (clean demo) and the original team notebook
data/raw/              BLS PDFs, UNRATE.csv, SHA256SUMS
results/               figures, metrics.json, report_scores.csv
docs/                  diagrams, security review, project slide deck, original figures
```

## Data quality fixes made during review

* **Date alignment.** The first version paired reports with unemployment by row position. FRED has no value for Oct-2025 (federal shutdown) and the Sept-2025 report was published in November, so the two series drifted apart. They are now joined on the reference month, with a unit test.
* **No label leakage.** The first classifier predicted a label derived from the same text, and crashed when every polarity was positive. The target is now the actual change in unemployment from FRED, tested on a time-ordered hold-out against a baseline.
* **Right text, faster.** Analysing only the narrative removes table vocabulary that dominated early word counts. PDF parsing went from about 16 minutes to 19 seconds, and dependencies are pinned.

## Limitations and next steps

* TextBlob is a general-purpose lexicon. A finance/economics lexicon (Loughran–McDonald) or a fine-tuned transformer is the natural upgrade.
* 121 documents (30 in the test window) means classifier numbers are indicative, not conclusive.
* Add CPI and payrolls as targets and test whether language *leads* the data with lagged correlations.
* Choose the number of LDA topics with coherence scores instead of fixing k = 4.

## Security

Dependency scan, data-handling notes and a hardening checklist are in [`docs/security_review.md`](docs/security_review.md).
