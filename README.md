# St. Lucia IMF Article IV: LLM extraction, RAG and dashboard

Coursework notebook for *Data Analysis with AI* (University of Hertfordshire). It reads the IMF staff report on St. Lucia, extracts structured macro-financial data with Gemini, checks every model output with a second model and with plain code, answers questions over the report with retrieval, and draws a dashboard.

## Which report, and why

Allocation: **St. Lucia, 2023-2024.** I read the period the same way as the brief's example (UK 2024-2025 maps to a report issued in 2025), so the report has to be *published* inside 2023-2024. In the IMF country report catalogue for St. Lucia, the latest Article IV staff report published in that window is:

- **IMF Country Report No. 24/68**, *St. Lucia: 2023 Article IV Consultation - Press Release; and Staff Report*, published 8 March 2024 (66 pages). This is the one used.
- IMF Country Report No. 25/65 (2024 Article IV) came out in March 2025, outside the window. If the period were read as the consultation year instead, only `pdf_name` and `pdf_url` in the settings cell need changing.

The PDF is in `data/stlucia_cr2468.pdf`. If it is missing, the notebook tries the IMF eLibrary link. The site often blocks scripted downloads, so in Colab it falls back to an upload prompt.

## What happens in the notebook

1. **PDF to clean text (Task 1).** `pypdf` extraction, repeated headers/footers/page numbers removed, broken lines and hyphenation joined, footnotes split off. The report's Segoe UI fonts have no character map, so `pypdf` drops some letters; those words are repaired from a Tesseract OCR pass, and the four tables printed as images (pages 33-36) are OCR'd. Sections come from the PDF bookmarks. Chunks are cut with the `bge-small-en-v1.5` tokenizer: at most 400 tokens, 50 tokens of overlap.
2. **Summaries and judge (Task 1).** Every section gets a summary under 100 words (word limit checked in code). The whole report gets an executive brief. A second, different Gemini model scores consistency, completeness and factual alignment claim by claim. Code checks that every number in a summary appears in its source. Flagged claims go back to the writer for one revision.
3. **Structured output (Task 2).** Theme density over the seven themes, measured three ways (keyword patterns, embedding similarity, LLM labels) and compared by Spearman rank correlation. 5-7 strengths and 5-7 vulnerabilities, each with a quote that is located on its page. The core indicator JSON in the exact target schema: every value has to appear in its quoted passage and fall inside a plausible range, or it becomes `null` with the reason. Time series from Table 1 and Table 5, each value checked against the printed table. The judge audits the numbers.
4. **RAG (Task 3).** The same bge vectors are put in a FAISS `IndexFlatL2`. Ten of my own questions are answered from the top 5 passages with chunk citations, next to a closed-book answer to the same question. The judge scores faithfulness, completeness and relevance, and code measures fact recall and citation validity. One question has no answer in the report, to test that the system declines.
5. **Extension: model and instruction benchmark.** Three Gemini variants × two instruction styles (strict system instruction with an enforced schema, or a plain user prompt) on the same indicator extraction, compared on schema compliance, accuracy against the verified values, grounding, latency and judge verdicts.
6. **Dashboard (Task 4).** Wide tables melted to long format with `pd.melt`. The dashboard has six panels plus an executive text panel and is saved at 300 dpi; each panel is also saved separately.

## Running it

**Colab.** Upload `StLucia_IMF_LLM_Pipeline.ipynb`, then *Runtime > Run all*. Set `repo_url` in the settings cell to this repository's URL so `data/` and `cache/` are cloned. Without a clone, upload `data/` and `cache/` into the Colab file panel first.

**Locally.**

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt          # and the tesseract-ocr system package
jupyter notebook StLucia_IMF_LLM_Pipeline.ipynb
```

## Models, keys and the cache

All model calls go through one gateway. Each answer is stored in `cache/llm_cache.sqlite` under a hash of model, system instruction, prompt and schema. The cache records which models produced it, and a rerun looks those answers up first. So **a rerun with the shipped cache makes no model calls and needs no key**. To prove this, set the environment variable `IMF_LLM_BACKEND=replay`: that mode refuses any live call.

Only answers missing from the cache go live, through the first backend available:
1. Colab's built-in Gemini (`google.colab.ai`, no key),
2. the Gemini API with `GEMINI_API_KEY` from Colab Secrets or the environment,
3. the Antigravity CLI (`agy`).

The writer and the judge are always different models. No key is written in the notebook or in this repository.

## Files

```
StLucia_IMF_LLM_Pipeline.ipynb    the notebook, with outputs
data/                             the IMF report
cache/llm_cache.sqlite            every model answer (replayed on rerun)
cache/ocr/                        OCR text of the pages that needed it
cache/chunk_vectors_*.npy         embedding vectors of the chunks
artifacts/json/                   every output as JSON (indicators, themes, RAG, judge audits, benchmark, call ledger)
artifacts/figures/                dashboard and single charts, 300 dpi
requirements.txt, LICENSE
```

## Known limits

- The judge is a Gemini model grading Gemini output, so some same-family leniency is possible. That is why each judge score sits next to a code check, and the benchmark measures how often the judge agrees with the verified values.
- Numbers read from the OCR'd image tables are checked against the OCR text only, not against a text layer. They are marked `verified_ocr` in `indicator_evidence.json`.
- St. Lucia has no national central bank. The policy rate is the ECCB's, which serves the whole currency union.

The written report is not part of this repository.

## Licence

MIT, see `LICENSE`.
