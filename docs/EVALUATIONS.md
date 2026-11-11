# Evaluations

Living document. One section per question rogo has been measured on. When a run finishes, replace the table it changes; do not add a second one. Dates are the day the numbers were taken.

Two test sets:

- **bench/multilingual**: public, 40 documents in five languages, 40 cross-lingual queries. Run with `bench/multilingual/run.py`. Rank 1 and MRR@5.
- **Vault set**: private, 48 hand-labelled questions over a personal note vault (36 hard, 12 older). The questions stay private. Scored as first place, top 3 and MRR of the first relevant note. One question is 2 to 3 points of top 3, so a gap of one or two questions is noise.

## Embedding models, bench/multilingual (26 Sep 2026)

| model | params | rank 1 | MRR@5 |
|---|---:|---:|---:|
| `gemma-q4` (default) | 300M | 35/40 | 0.927 |
| `bge-m3-gguf` | 568M | 34/40 | 0.904 |
| `qwen3-gguf` | 600M | 31/40 | 0.851 |
| `granite-gguf` | 278M | 28/40 | 0.781 |

CI keeps the default at 34/40 and MRR@5 0.90 or better (`cargo e2e`).

## Learned sparse lane and small dense models (started 10 Nov 2026)

**Question.** A public forum thread reported that a SPLADE-style learned sparse model beat Qwen3-Embedding-0.6B on the poster's own Korean corpus, and that fusing it with dense was worse than sparse alone. Does a learned sparse lane, or a smaller dense model, beat the current BM25 + EmbeddingGemma on the vault set?

**Status, 11 Nov 2026.** Dense models done, sparse models running. Verdict open. No engine change until a variant wins on both sets.

**Method.** A Python prototype, outside this repo, on CPU. The vault is cut into 1,800-character chunks with 200 overlap and 512 tokens at most; a note ranks by its best chunk. Baseline lanes come from `rogo --lex` and `--sem`, fused as `--hybrid` fuses them (CombMAX, k = 3, lexical weight 1.05 with terms). Each candidate lane is scored alone, fused in place of EmbeddingGemma, and for sparse models also as a third lane and in place of BM25.

### Dense models alone, vault set, all 48

| model | params | first | top 3 | MRR |
|---|---:|---:|---:|---:|
| EmbeddingGemma, default | 300M | 33 | 37 | not kept |
| granite-embedding-97m-multilingual-r2 | 97M | 30 | 38 | 0.725 |
| gte-small | 33M | 29 | 35 | 0.686 |
| arctic-embed-s | 33M | 28 | 35 | 0.676 |
| multilingual-e5-small | 118M | 25 | 36 | 0.652 |

No small model beats the default on first place. granite-97m-r2 matches it on top 3 at a third of the size, so it is the one to test fused.

### Sparse models

Candidates: KURE-Sparse-nano, opensearch-neural-sparse-encoding-multilingual-v1, mSlideOn, SlideOn. Results pending.

| variant | first | top 3 | MRR |
|---|---:|---:|---:|
| `--hybrid` today | pending | pending | pending |
| BM25 + EmbeddingGemma + mSlideOn | pending | pending | pending |
| mSlideOn replacing BM25 | pending | pending | pending |

### What would change rogo

A variant has to beat today's fused score on the vault set and keep bench/multilingual at 34/40 or better. A sparse lane also needs a route into the engine: SPLADE is a BERT masked-language-model head, which llama.cpp may not run. Cost to report with a win: index size, indexing time, query latency, memory.

### Not tried

Models above about 600M, a reranker (rejected 20 Oct 2026, at most 1 more first place of 48), fine-tuning.

## Cross-encoder reranker (20 Oct 2026)

Rejected: at most 1 more first place of 48.
