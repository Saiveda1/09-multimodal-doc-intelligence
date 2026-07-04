# Document Intelligence Project Document

**Prepared For:** Sai Veda  
**GitHub Publishing Account:** Nikeshk834  
**Repository Slug:** `09-multimodal-doc-intelligence`  
**Verified Test Count From Portfolio Index:** 18  

## Background

**Layout-aware document understanding for invoices, receipts, purchase orders and
forms — at scale.** Turn a page of tokens-with-boxes into structured, typed
fields: identifiers, dates, parties, totals, and the full line-item **table** —
then route the document and make its content searchable. Fully offline,
deterministic, no GPU, no paid APIs.

```
 ┌── document (tokens: text + (x,y,w,h) + gold label) ──┐
 │  generator ▸ features ▸ TokenClassifier ▸ field types │
 │                                    │                   │
 │            router ▸ doc type   group_entities          │
 │            embed  ▸ search/dedup   ├─ pair_key_values  │  ← geometry
 │                                    └─ extract_table    │  ← row/col clustering
 └───────────────────────────────────────────────────────┘
```

## Headline results

Measured on a held-out set of **2,000 documents / 98,687 tokens** (train: 8,000
docs / 396,586 tokens), seed-fixed and reproducible with `make run`:

| Metric | Value |
|---|---|
| **Field extraction macro-F1** (token typing) | **0.976** |
| Field micro-F1 / token accuracy | 0.985 / 0.986 |
| Majority-class baseline (token acc) | 0.189 → model is **5.2×** |
| **Doc-type classification accuracy** (router) | **1.000** |
| **Line-item table cell accuracy** (end-to-end) | **1.000** on 1,582 tables |
| Key→value linkage accuracy | 0.964 on 11,181 key/value pairs |
| **Inference throughput** | **1,571 docs/s · 77k tokens/s** (single process) |

Corpus actually generated: **100,000 documents = 4,962,503 tokens** streamed to a
**18.8 MB** zstd Parquet token table, queried out-of-core with DuckDB.

## Project Purpose

This repository is part of the AI engineering portfolio and focuses on the following problem space:

- Layout-aware field + table extraction
- Headline result from the portfolio index: field macro-F1 **0.976**; 100k docs, doc-type **100%**

## What This Project Solves

This project provides a production-style implementation with benchmark evidence and operational checks committed into the repository.

## Technical Approach

# Architecture & Design Decisions

## 1. Problem

Enterprise document AI (accounts-payable, KYC, claims) has to turn a page image
into **structured, typed fields**: which tokens are the invoice number, the
date, the vendor, the totals, and the line-item table — then route the document
and make its content searchable. The hard part is that the signal is **spatial**:
`"$1,240.00"` is a `TOTAL` in the bottom-right totals block and an `LI_AMOUNT`
inside the table. A layout-blind text model cannot tell them apart.

## 2. Why synthetic *rendered layouts*

There is no offline OCR/vision stack here, so we generate documents as
**structured layouts**: every document is a list of word-level tokens, each with
`(text, x, y, w, h)` and a gold label. This is a standard, legitimate way to
prototype document AI (FUNSD/CORD/DocILE all expose exactly this token+box+label
schema) and it buys two things a scanned corpus cannot:

* **Perfect, controllable ground truth** at arbitrary scale — labels, table row
  groups, and key→value structure are known by construction, so every metric is
  exact and every failure is attributable.
* **A real image** — `render.py` rasterizes the layout to PNG with matplotlib,
  so the "money-shot" screenshot is an actual rendered document with detected
  boxes overlaid, not a mockup.

The pipeline consumes a generic `(text, x, y, w, h)` token stream, so a real
backend (Tesseract words, or LayoutLM/Donut token embeddings) drops in at the
`documents_to_table` boundary **without touching** feature extraction, the
classifier, KV pairing, table recovery, routing, or evaluation. See README §
"Swapping in a real OCR/vision model".

## 3. Pipeline

```
 generator ──► token table ──► features ──► TokenClassifier ──► field labels
 (text,box,      (columnar)     (pos+text+     (SGD softmax)          │
  label)                         neighbour)                           ├─► group_entities
                                                                      │      │
                     router (RandomForest) ──► doc type / route       │      ├─► pair_key_values (geometry)
                                                                      │      └─► extract_table (row/col clustering)
                     embedding (TF-IDF+SVD) ──► search / dedup ◄───── extracted content
```

### 3.1 Feature extraction (`features.py`)
Every token becomes a dense vector from

## Benchmark And Validation Evidence

The portfolio root documents **18 passing tests** for this project, and the repo quickstart uses `make test` as the standard validation path. The benchmark outputs committed in `benchmarks/` and the generated visuals in `assets/` are the evidence package for this delivery.

### results.md

# Scaling benchmark

Streaming generation (bounded memory) + extraction inference.

| Docs | Tokens | Gen docs/s | Gen tokens/s | Infer docs/s | Infer tokens/s |
|---|---|---|---|---|---|
| 2,000 | 99,071 | 728 | 36,065 | 1,838 | 90,840 |
| 10,000 | 497,136 | 1,027 | 51,037 | 2,262 | 111,818 |
| 50,000 | 2,482,852 | 774 | 38,452 | 2,108 | 104,170 |
| 100,000 | 4,964,026 | 1,009 | 50,080 | 2,405 | 118,876 |

Peak sustained generation: **51,037 tokens/s** (1,027 docs/s), single process, bounded memory.

Projected wall-clock to stream **1B tokens**: **5.4 h** single-process; near-linear with shards (generation is a pure function of `(seed, doc_id)`).

## Visual Artifacts Reviewed

- `assets/detected_document.png`: Detected fields on a rendered document (the product view).
- `assets/field_f1.png`: Per-field precision / recall / F1.
- `assets/doc_type.png`: Document-type router: confusion matrix + routed mix.
- `assets/dashboard.png`: Extraction KPIs + throughput scaling.

## Engineering Notes

The primary design and scale decisions are documented in [`ARCHITECTURE.md`](./ARCHITECTURE.md). The benchmark markdown in [`benchmarks/`](./benchmarks) and the generated figures in [`assets/`](./assets) should be read together: the markdown gives the measured numbers, and the screenshots make those results easier to inspect quickly during review.

## Files Included In This Repo

- [`README.md`](./README.md) for project overview, quickstart, and headline results
- [`ARCHITECTURE.md`](./ARCHITECTURE.md) for system design and scaling choices
- [`benchmarks/`](./benchmarks) for measured results from the committed runs
- [`assets/`](./assets) for generated screenshots and dashboards
- [`tests/`](./tests) for the automated validation suite

## Delivery Summary

This project document was prepared for **Sai Veda** so the repository reads like a real project handoff: what the system is for, what problem it solves, what evidence supports it, and where the benchmark and test artifacts live inside the repo.
