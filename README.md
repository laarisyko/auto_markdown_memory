# auto_markdown_memory

AutoMem turns a folder of Markdown files into an external memory for LLMs.

You drop sources into `raw/`. A Research Agent reads them and writes compressed,
linked notes into `memory/`. Fresh Evaluator Agents then answer questions using
only what a fixed retrieval protocol surfaces from `memory/`, and Judge Agents
score those answers against gold answers in `eval/`. Patches that improve
quality without regressing anything are committed; everything else is rolled
back.

The full spec — agent roles, retrieval protocol, scoring rule, epoch lifecycle —
lives in [`AutoMem.md`](AutoMem.md). This README is just the map.

## Folder layout

```text
auto_markdown_memory/
  AutoMem.md      # the spec
  README.md       # you are here
  raw/            # ground-truth sources you provide; never edited
  memory/         # flat folder of Markdown notes; the external memory
                  # seed file is memory/INDEX.md
  eval/
    questions/    # current epoch's questions (Q001.md, ...)
    gold/         # current epoch's gold answers (Q001.gold.md, ...)
    replay/       # questions + gold that persist across epochs
      questions/
      gold/
    runs/         # per-patch baseline / after / decision artifacts
    epochs/       # archived past epochs (questions, gold, splits, summary)
```

## How to use it

1. Put source material into `raw/`. Treat it as read-only after that.
2. Let the Research Agent read `raw/` and seed `memory/INDEX.md` plus a few
   linked notes.
3. At each epoch start, a Question-Writing Agent generates fresh
   `eval/questions/` and `eval/gold/` from `raw/` (without reading `memory/`).
4. Run the per-patch training loop: baseline → patch `memory/` → re-evaluate
   with fresh Evaluator + Judge agents → accept or roll back.
5. At epoch end, re-score the `eval/replay/` set; if it holds, tag
   `memory_epoch_N` and optionally promote a few questions into replay.

## Rules of thumb

- `raw/` is ground truth and never edited.
- `memory/` is Markdown only, no subfolders — topology lives in `[[wikilinks]]`.
- Evaluator Agents never see `raw/` or `eval/`. If they can't answer from what
  retrieval surfaces, that's a training failure (missing content or bad links).
- Quality first, tokens second. Hallucination must never increase.
- Memory only improves through committed patches.
