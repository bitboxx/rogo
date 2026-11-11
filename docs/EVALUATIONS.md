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

**Status, 11 Nov 2026.** Done on the vault set. **No variant beats today's BM25 + EmbeddingGemma `--hybrid`**; the best ties it. No engine change. bench/multilingual was not run, since every variant failed the first test.

**Method.** A Python prototype, outside this repo, on CPU. The vault is cut into 1,800-character chunks with 200 overlap and 512 tokens at most; a note ranks by its best chunk. Baseline lanes come from `rogo --lex` and `--sem`, fused as `--hybrid` fuses them (CombMAX, k = 3, lexical weight 1.05 with terms). Each candidate lane is scored alone, fused in place of EmbeddingGemma, and for sparse models also as a third lane and in place of BM25.

### Dense models alone, vault set, all 48

| model | params | first | top 3 | MRR |
|---|---:|---:|---:|---:|
| EmbeddingGemma, default | 300M | 33 | 37 | not kept |
| granite-embedding-97m-multilingual-r2 | 97M | 30 | 38 | 0.725 |
| gte-small | 33M | 29 | 35 | 0.686 |
| arctic-embed-s | 33M | 28 | 35 | 0.676 |
| multilingual-e5-small | 118M | 25 | 36 | 0.652 |

No small model beats the default on first place alone. Fused, none helps either (next table).

### Learned sparse models alone, vault set, all 48

| model | first | top 3 | MRR |
|---|---:|---:|---:|
| mSlideOn | 29 | 40 | 0.721 |
| SlideOn | 27 | 37 | 0.699 |
| opensearch-neural-sparse multilingual v1 | 28 | 35 | 0.677 |
| KURE-Sparse-nano (the model in the thread) | 24 | 31 | 0.612 |

mSlideOn alone has the best top 3 of any single lane here (40; BM25 with terms 41, EmbeddingGemma 37) but a weak first place.

### Fused, vault set, all 48 (first / top 3 / MRR)

Today's `--hybrid` on the same run: **31 / 43 / 0.781**. BM25 alone 33 / 41 / 0.783, EmbeddingGemma alone 33 / 37 / 0.758.

| variant | first | top 3 | MRR |
|---|---:|---:|---:|
| any small dense model replacing EmbeddingGemma | 31 | 41-42 | 0.764-0.775 |
| any small dense model as a third lane | 31 | 42 | 0.770-0.774 |
| any sparse model replacing EmbeddingGemma | 31 | 41 | 0.763-0.773 |
| any sparse model as a third lane, weight 1.0 | 31 | 42 | 0.771-0.775 |
| any sparse model as a third lane, weight 0.5 or 0.25 | 31 | 43 | 0.781-0.782 |
| sparse model replacing BM25 | 33 | 38-39 | 0.758-0.769 |

A third lane at full weight costs one top-3 hit; at 0.5 or 0.25 it changes nothing. Replacing BM25 gains two first places and loses four to five top-3 hits. All gaps are within a few questions of 48, so they are noise, and none is a win. Taken from the best of each group: mSlideOn as a third lane at 0.5 scores 31 / 43 / 0.781, identical to today.

The thread's claim that fusing sparse with dense is worse than sparse alone does not apply here: the fused scores sit above every sparse lane alone on top 3 except mSlideOn's 40, and equal to it at the 0.5 weight.

Baseline caveat: this prototype scores today's `--hybrid` at 31 / 43 on the 48, below the 38 / 45 recorded on 17 Oct 2026, because the vault has grown since and the prototype runs rogo's lanes without the earlier per-question tuning. Compare variants with each other and with this run's baseline, not with the October figures.

### What would change rogo

Nothing in these runs. A variant would have to beat 31 / 43 / 0.781 on the vault set and keep bench/multilingual at 34/40 or better. A sparse lane would also need a route into the engine (SPLADE is a BERT masked-language-model head, which llama.cpp may not run), and the cost in index size, time and memory: encoding the 17,205 chunks took 17 to 22 minutes per sparse model on CPU, and each document held about 450 to 1,430 non-zero terms.

### Worth trying next

mSlideOn is the only candidate with a distinct signal (best top 3 alone). The open question is whether it rescues queries both current lanes miss; a per-question overlap check against today's two lanes would answer that without more encoding.

### Not tried

Models above about 600M, a reranker (rejected 20 Oct 2026, at most 1 more first place of 48), fine-tuning.

## Cross-encoder reranker (20 Oct 2026)

Rejected: at most 1 more first place of 48.
