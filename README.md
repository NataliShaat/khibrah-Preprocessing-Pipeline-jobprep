# jobprep — Khibrah job-posting pre-processing pipeline

One pipeline for every job posting the project will see: the historical JobArabi
export (Meltwater, X/Twitter) today, and scraped job-board postings in GP2. It
handles Arabic, English, French and mixed Arabic/English text without any
per-language branching, and depends only on `pandas`, `numpy` and `openpyxl`
(all preinstalled in Colab).

## Quick start (Colab)

```python
!unzip -q khibrah-jobprep.zip -d /content && pip install -q -e /content/khibrah-jobprep
from jobprep import JobPostingPipeline

pipe = JobPostingPipeline()
result = pipe.run_file("jobarabi_combined.csv")   # adapter is detected automatically
result.report                                      # counts per step, coverage, languages
result.unique                                      # model-ready rows
result.save("processed/")                          # CSV (+ parquet with formats=("csv","parquet"))
```

Command line: `python -m jobprep jobarabi_combined.csv --out processed/`

## What it does

Each step is a public method on `JobPostingPipeline`, so the notebook runs them one at a time. `run()` calls them in this order:

| # | Method | What happens | Rows |
|---|--------|--------------|------|
| 0 | `drop_null_columns` | Raw columns ≥ 95% missing are dropped (13 of 47 in JobArabi) | columns dropped |
| 1 | `adapt` | Source columns → one canonical schema | – |
| 2 | `remove_duplicate_records` | Same record id or URL captured twice | **removed** |
| 3 | `parse_dates` | UTC timestamp, Riyadh date and month; relative dates ("3 days ago", "منذ 3 أيام") resolved | – |
| 4 | `remove_missing` | No text, or no usable posting date; optional fields stay empty, never imputed | **removed** |
| 5 | `clean` | `text_clean`: URLs, mentions, emails, emojis, HTML, tatweel, invisible chars removed; hashtags → words; Arabic-Indic digits → 0-9 | – |
| 6 | `remove_invalid_text` | Placeholders ("This account doesn't exist") and texts < 15 characters | **removed** |
| 7 | `detect_language` | `lang` ∈ ar / en / fr / mixed / latin / unknown | – |
| 8 | `normalize` | `text_norm`: diacritics removed, أ/إ/آ→ا, ى→ي, ة→ه, Persian letters folded, Latin lower-cased and de-accented, punctuation removed (C++, C# kept) | – |
| 9 | `find_locations` | Saudi and foreign cities and countries named in the text | – |
| 10 | `filter_relevance` | Job seekers, scam warnings, course ads, labour news → `is_job_posting=False` | flagged |
| 11 | `filter_scope` | Country from text first, then source metadata → `job_country`, `scope_basis`; outside Saudi Arabia → `in_scope=False` | flagged |
| 12 | `deduplicate` | Exact + near-duplicate reposts within 30 days → `is_duplicate=True`; earliest copy kept | flagged |

`result.clean` = rows with `is_job_posting & in_scope & ~is_duplicate`, keeping only content, date, source and place. `result.step_log` gives rows before/after each step.

On JobArabi: 16,465 raw rows → 387 removed → 16,078 kept → 10,363 in Saudi Arabia → **6,542 unique Saudi job advertisements** in the clean file.

### Country scope rule

1. A city or country named in the text (or in a scraped `location` field) decides. A post naming Saudi Arabia and another country stays in scope.
2. If the text names no place, the source location is used. For X/Twitter this is the author's profile country, a weaker signal, recorded as `metadata_target`.
3. With no evidence at all the post is excluded (`cfg.scope.keep_unknown`).

`cfg.scope.target_country = None` keeps every country.

### Charts

`jobprep.plots` draws every notebook chart in the Khibrah palette only: PNU Teal `#00797F`, Plum `#5B3A6E`, Burgundy `#7A2E3B`. Bars carry direct value labels because Plum and Burgundy are close in lightness. The monthly chart marks the highest and lowest months with Burgundy dots.

## Outputs of `result.save()`

* `postings_clean.csv`: **the file for skill extraction.** One row per unique Saudi job advertisement with
  `posting_id`, `source`, `source_url`, `posted_date_local`, `posted_month`, `job_country`, `cities_mentioned`,
  `lang`, `text_clean`, `text_norm`. Columns empty for the whole source are dropped automatically.
* `postings_all.csv`: the audit file. Every kept row with every flag and reason, to show why a row was excluded.
* `rejected.csv`: removed rows with `invalid_reason` (duplicate_record, missing_text, missing_date, placeholder_text, too_short).
* `report.json`: counts per step, dropped columns, language and country distribution, full config.

`text_clean` keeps the original spelling and is what an LLM reads. `text_norm` folds spelling variants and is
what keyword or ontology matching uses.

## Using it on scraped postings (GP2)

Column names are matched to common synonyms automatically (`title`, `job_title`,
`description`, `requirements`, `company`, `location`, `date_posted`, `url`, ...,
plus Arabic names such as `المسمى الوظيفي`).

```python
from jobprep import JobPostingPipeline, DedupIndex

pipe = JobPostingPipeline()
index = DedupIndex.load("dedup_index.pkl")          # everything seen so far

batch = pipe.run_records(scraped_records, source="bayt", index=index)
index.save("dedup_index.pkl")                        # new postings are now "seen"

one = pipe.process_one(record, source="linkedin", index=index)   # online, one posting
```

For a source with unusual fields, map them explicitly:

```python
from jobprep import MappingAdapter
adapter = MappingAdapter(
    {"source_record_id": "jid", "title_raw": "header",
     "body_raw": ["summary", "duties", "requirements"],   # concatenated
     "company_raw": "org", "posted_at_raw": "published"},
    defaults={"source": "my_site", "source_is_job_board": True},
)
result = pipe.run(df, adapter=adapter)
```

## Configuration

All thresholds live in `jobprep/config.py`:

```python
from jobprep import PipelineConfig, JobPostingPipeline
cfg = PipelineConfig()
cfg.dedup.window_days = 14          # reposts > 14 days apart count as new demand
cfg.cleaning.remove_emojis = False
cfg.normalization.normalize_taa_marbuta = False
pipe = JobPostingPipeline(cfg)
```

Lexicons (education levels, cities, currencies, relevance cues, ...) live in
`jobprep/lexicons.py` and can be extended without code changes.

## Tests

`pytest tests/` runs 25 scenario tests: Arabic, English, French and mixed posts,
HTML descriptions, placeholders, emoji-only posts, scam warnings, job seekers,
cross-batch de-duplication, the repost window, null-column dropping, missing dates and the country scope.

## Known limits

* Language labels are script-based; Arabizi (Arabic written in Latin letters) is labelled `en`/`latin`.
* For X/Twitter, posts that name no place fall back to the author's profile country (`scope_basis = metadata_target`), a weaker signal than a place named in the text.
* Posts that link to an article without describing the job are kept but carry little text.
