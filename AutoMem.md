# AutoMem

AutoMem trains a folder of Markdown files into an external memory for LLMs.

## Core summary

```text
project/
  AutoMem.md
  raw/      = user-provided source files; ground truth; never edited
  memory/   = flat folder of Markdown files; the external LLM memory
  eval/     = questions, gold evidence, run reports; NOT visible to evaluators
```

The Research Agent reads `raw/`, `memory/`, and `eval/` to improve `memory/`. Fresh Evaluator Agents answer questions using only what a fixed retrieval protocol surfaces from `memory/`. Fresh Judge Agents score answers against `eval/gold/`.

The agent optimizes two things together: the **content** of `memory/*.md` and the **link topology** connecting them. Topology is part of training, not a hidden variable.

Acceptance is lexicographic and quality-first:

1. No quality dimension regresses beyond the noise floor (hallucination must never increase at all).
2. At least one quality dimension improves OR total tokens decrease.
3. Subject to (1) and (2), prefer patches that reduce tokens.

Every accepted patch must be committed and pushed to git. Memory improves only through commits.

---

## Core idea

AutoMem treats memory optimization like training.

Normal LLM training:

```text
raw data -> training -> model weights -> better answers
```

AutoMem:

```text
raw sources -> memory training -> Markdown files + link topology -> better answers by fresh LLMs
```

The Research Agent improves `memory/` so that fresh LLM instances can answer questions that would otherwise require reading `raw/`. AutoMem trains the memory, not the model.

---

## Folder roles

### `raw/`

User-provided source files. Ground truth. Examples: chat exports, PDFs, transcripts, architecture docs, decision logs.

Rules:

- `raw/` is read-only. Never modify, delete, or rewrite.
- Use `raw/` as evidence for training and judging memory.
- If a claim cannot be supported by `raw/`, mark it uncertain or omit it.

### `memory/`

Flat folder of Markdown files. The external memory.

Rules:

- Markdown files only.
- No subfolders. Topology is encoded in `[[wikilinks]]`, not directories.
- Every file should be useful to both humans and fresh LLMs.
- Files must be concise, structured, source-grounded, retrieval-friendly.
- The link graph replaces folder hierarchy and IS optimized.
- One file is the seed: `memory/INDEX.md`. The retrieval protocol always starts here.
- Do not copy `raw/` into `memory/`. Compress and structure.

### `eval/`

Evaluation artifacts. Visible to Research Agent and Judge Agent. **Never visible to Evaluator Agents.**

```text
eval/
  questions/           # current epoch's train / validation / test
    Q001.md
    Q002.md
    ...
  gold/                # current epoch's gold answers
    Q001.gold.md
    ...
  replay/              # accumulates ACROSS epochs; the anti-forgetting anchor
    questions/
    gold/
  splits.yaml          # train / validation / test for the current epoch
  noise_floor.yaml     # estimated per-dimension judge stddev
  runs/
    2026-04-12_run17/  # one dir per training run (multiple per epoch)
      baseline.json
      after_patch.json
      decision.md
  epochs/              # archive of past epochs (questions, gold, splits, summary)
    epoch_4/
      questions/
      gold/
      splits.yaml
      summary.md
  metrics_history.csv
```

Rules:

- Research Agent may read everything in `eval/` *except* test-split questions and gold during patch design. Test material is opened only for periodic test runs.
- Judge Agent may read the relevant gold file for the question being judged.
- Evaluator Agent receives only the question text. Nothing else from `eval/`.
- Commit `eval/`. It is part of the project.

`eval/` exists as a separate folder because gold answers, expected elements, and run reports cannot live in `memory/` (the evaluator would see them) and do not belong in `raw/` (they are synthesized artifacts, not ground truth).

---

## Epochs

Training happens in discrete **epochs**, each triggered by meaningful growth in `raw/`. The analogy is sleep: each night the brain consolidates new experiences into long-term memory, but doesn't continuously regenerate its own training signal. Within an epoch, training is self-contained; across epochs, only protected memories (replay) carry forward.

### When to start a new epoch

Start a new epoch when `raw/` has been extended with material that introduces new concepts, decisions, entities, or events that the existing memory has no representation of. A single new transcript that just adds detail to known topics does NOT warrant an epoch — handle it with normal patches against the current epoch's question set.

Rough trigger heuristics (any one is sufficient):

- New `raw/` files contain entities (people, customers, projects) not yet in `memory/`.
- A category of decisions or events appears that wasn't covered before.
- Cumulative new `raw/` content exceeds ~20% of prior `raw/` size.
- A user explicitly requests a fresh epoch.

A normal patch loop handles minor `raw/` extensions. Epoch boundaries are for structural change.

### Epoch lifecycle

```text
1. Trigger: raw/ has grown meaningfully.
2. Snapshot: git tag memory_epoch_N (the starting checkpoint).
3. Generate: a fresh Question-Writing Agent reads ALL of raw/ (old + new)
   and writes train/validation/test from scratch into eval/questions/
   and eval/gold/. The Question-Writing Agent must NOT read memory/.
4. Carry: replay set is unchanged from epoch N-1 (eval/replay/ is untouched).
5. Train: run the normal per-patch training loop until validation
   saturates or the patch budget is exhausted.
6. Verify: re-score the carried-forward replay set against the new memory.
7. Decide:
   - If replay regressed beyond noise:  EPOCH FAILED → roll back to
     memory_epoch_N (`git reset --hard memory_epoch_N`).
   - If replay held: EPOCH SUCCEEDED → tag memory_epoch_(N+1).
8. Promote: optionally promote critical validation questions from this
   epoch into eval/replay/ for future epochs. Promote sparingly.
9. Archive: move this epoch's questions/gold/splits into eval/epochs/N+1/
   along with a short summary.md (size of raw delta, new entities covered,
   final metrics, regressions, promotions to replay).
```

### Why train/validation/test regenerate but replay accumulates

Train, validation, and test are generated fresh per epoch because:

- New `raw/` material produces new failure modes that need new probes.
- Old questions designed against old `raw/` may no longer reflect what users will ask.
- Within one epoch, the question set is stable and patch decisions are comparable.
- Across epochs, scores are NOT directly comparable — different question sets, different difficulty distributions. Cross-epoch progress is tracked through meta-properties (memory size, retrieval budget usage, link-graph navigability), not raw quality scores.

Replay accumulates because:

- It is the anti-forgetting anchor.
- A regression on a year-old question matters even if no current-epoch question covers that topic.
- Without persistent replay, each epoch could quietly forget what earlier epochs taught.
- Replay should grow slowly: only critical questions, only when they have clear long-term value. Bloated replay slows every future epoch.

### What persists across epochs

| Survives | Resets per epoch |
|----------|------------------|
| `memory/` (modified by training, tagged at boundaries) | `eval/questions/` |
| `eval/replay/` (accumulates by promotion) | `eval/gold/` |
| `eval/noise_floor.yaml` (re-estimate when judge model changes) | `eval/splits.yaml` |
| `eval/epochs/N/` archives of past epochs | New `eval/runs/` per epoch |

### A failed epoch is OK

Some `raw/` extensions damage memory faster than they improve it. The replay verification step catches this. A failed epoch rolls back cleanly because `memory_epoch_N` is a git tag.

The system is allowed to wake up worse than it went to sleep — but only if it then reverts to yesterday and tries again with smaller patches or a different patch strategy. A failed epoch should be archived in `eval/epochs/N+1_failed/` with a `summary.md` explaining what broke, so future epochs don't repeat the same mistake.

---

## Objective and scoring

### Quality is a vector

For each evaluation, the Judge returns five dimensions, each in `[0, 1]`:

```text
correctness        - are the stated facts true per raw/?
completeness       - does the answer cover the expected elements?
hallucination      - 1.0 = no fabrications; 0.0 = mostly fabricated
calibration        - appropriate uncertainty, abstention, time-scoping
source_consistency - consistent with the rest of memory/
```

These are reported as a tuple, not averaged. Different failure modes need different responses.

### Token cost

```text
total_tokens = retrieval_input_tokens + answer_output_tokens
```

`retrieval_input_tokens` is the size of the context produced by the retrieval protocol, plus the question. It depends on both the content of files and the topology that determines what gets surfaced.

### Acceptance rule (lexicographic, quality-first)

A patch is accepted only if **all** of these hold on the validation batch:

1. **No dimension regresses beyond noise.**
   - `hallucination` must not decrease at all. Memory must never start lying.
   - For other dimensions: mean must not drop by more than `max(0.02, 2 * noise_stddev)`.
2. **No single-question drop exceeds `max(0.05, 2 * noise_stddev)` on any dimension.**
3. **Critical / replay questions do not regress at all** beyond noise.
4. **At least one of:** some quality dimension improves; OR total tokens decrease. (Otherwise the patch is a no-op and is rejected for cost reasons alone.)
5. **The patch only modifies `memory/*.md`** and respects the patch-size cap (see below).

Subject to (1)–(5), prefer the patch with the largest token reduction.

This replaces the old `score = quality / tokens` formula. That formula rewards short mediocre answers because tokens are unbounded while quality is in `[0, 1]`. Lexicographic ordering with explicit thresholds enforces "quality first" structurally instead of as a slogan.

---

## Retrieval protocol

The Evaluator Agent does NOT receive all of `memory/` at once. It receives whatever a fixed retrieval procedure surfaces. The Research Agent optimizes both file content and link topology under this procedure.

### Default protocol: link-walk from seed

```text
1. Start at memory/INDEX.md.
2. Read INDEX.md in full.
3. Pick the most relevant [[wikilinks]] given the question + INDEX content.
4. Read each chosen file. Extract its links. Add unread ones to a queue.
5. Continue breadth-first to max_depth = 3.
6. Stop when accumulated retrieval tokens reach budget B (default 4000).
7. The collected file contents become the Evaluator's context.
```

Why this matters: topology becomes a first-class optimization target.

- A fact in a file unreachable from `INDEX.md` is invisible to the evaluator.
- Adding a link can surface a fact without duplicating content.
- Removing a link can prune low-value content from the retrieval window.
- Restructuring `INDEX.md` changes what is reachable in K hops.
- A bloated `INDEX.md` exhausts the budget before useful files load.

### Protocol invariants

- The retrieval protocol is **fixed within a run.** Parameter changes (seed, depth, budget, ranker) are themselves checkpoints — they make scores incomparable across versions.
- The protocol must be **deterministic** given the question and `memory/` state, so metric changes can be attributed to memory edits.
- Alternative protocols (full-folder load, BM25, embedding-based) are valid. Pick one and document it in `eval/splits.yaml` so runs are reproducible.

---

## Fresh-agent principle

Four roles, organized by scope.

**Per-epoch (runs once at epoch start):**

- Question-Writing Agent

**Per-training-loop (runs every patch cycle within an epoch):**

1. Research Agent
2. Evaluator Agent
3. Judge Agent

### Question-Writing Agent

Fresh instance. Runs once at the start of each epoch.

May read: `raw/` in full (old + new). May read prior epochs' archived questions in `eval/epochs/N/` for de-duplication only — to avoid writing near-identical questions to ones the system has already answered.

May NOT read: `memory/`. This is critical. If the question writer reads memory, it will (consciously or not) write questions memory already handles well, or avoid weaknesses that ought to be surfaced. The point of fresh question generation is to probe what's in `raw/`, not to validate what's in `memory/`.

May NOT read: `eval/replay/`. Replay is held back from the question writer so new questions can't accidentally duplicate replay coverage.

Produces: `eval/questions/`, `eval/gold/`, `eval/splits.yaml` for the new epoch. Includes the abstention quota (~15–25% of the batch).

**Two-eyes principle for gold answers:** one fresh instance writes the gold reading `raw/`; a second fresh instance verifies that the cited `raw/` evidence actually supports the gold's required elements. Cheap insurance against fabricated gold, which would corrupt every downstream metric.

### 1. Research Agent

May read: `raw/`, `memory/`, validation/replay questions and gold, prior `eval/runs/`, archived `eval/epochs/`.
May NOT read during patch design: test-split questions, test-split gold.
May edit: `memory/*.md` only.
May NOT edit: `raw/`, gold answers, splits.

### 2. Evaluator Agent

Fresh instance per evaluation. Used for validation, replay, and held-out test questions. Same role, different inputs.

May read: the question text, and whatever the retrieval protocol surfaces from `memory/`.
May NOT see: `raw/`, `eval/`, gold, expected elements, patch diffs, the question's split, prior answers, Research Agent rationale.

If the Evaluator cannot answer from retrieved memory, either the memory does not contain the information, or the topology does not surface it. Both are training failures.

### 3. Judge Agent

Fresh instance per evaluation. Stochastic — multiple runs are required (see noise handling).

May read: the question, the Evaluator's answer, the corresponding `eval/gold/Q*.gold.md` (or `eval/replay/gold/` for replay questions).
Returns: the five-dimension quality vector.

---

## Evaluation noise handling

LLM-as-judge scoring is stochastic. The same answer scored twice can drift by 0.05–0.10 — exactly the size of naive acceptance thresholds. Without noise handling, patches are accepted and rejected by dice rolls.

Required practice:

- **N = 3 reruns** for both Evaluator answer generation and Judge scoring. Report mean and stddev per dimension.
- **Estimate the noise floor once per major model change.** Run an unchanged baseline 10× across the validation set. The per-dimension stddev becomes `noise_stddev`, stored in `eval/noise_floor.yaml`.
- **Acceptance thresholds use `max(<static threshold>, 2 * noise_stddev)`.** A "regression" only counts if it exceeds 2σ.
- **Cheaper option:** temperature 0 + N = 2. Strictly better than N = 1.

Without this, ship/no-ship decisions are statistically indistinguishable from coin flips at the margin.

---

## Training loop (per-patch, within an epoch)

This loop runs repeatedly within a single epoch. The epoch lifecycle (see Epochs above) wraps it: the Question-Writing Agent has already produced this epoch's questions before Step 1 here, and the replay verification + epoch tagging happens after the last patch of the epoch.

### Step 1: Start from clean git

```bash
git status
```

Current git HEAD is the current memory checkpoint. Do not start a run with unrelated uncommitted changes.

### Step 2: Inspect inputs

Read or sample `raw/`. Review existing `memory/`. Review prior `eval/runs/` for failure patterns. Look for durable knowledge: decisions, timelines, customers, incidents, terminology, architecture, ownership, open questions, resolved debates.

### Step 3: Confirm the question set

For the FIRST patch of an epoch: questions and gold already exist in `eval/questions/` and `eval/gold/`, written by the Question-Writing Agent at epoch start. The Research Agent does NOT modify them.

For SUBSEQUENT patches in the same epoch: same question set, unchanged. Stability of questions within an epoch is what makes patch-level metric comparisons valid.

If the Research Agent believes a question is malformed or the gold is wrong, it flags it for human review rather than editing it. Editing eval material mid-epoch invalidates the metric.

Question file format:

```md
# Q042
## Question
Why did the team stop using Approach X for Customer A?
## Tags
customer-a, decision, deprecated-approach
```

Gold file format:

```md
# Q042 expected
## Required elements
- Mentions the performance issue observed in March 2023.
- Notes that the original workaround was temporary.
- Mentions Approach Y as the replacement.
## Forbidden claims
- Do NOT generalize the decision to all customers.
## Calibration
- Should note uncertainty if memory does not show current status.
## Evidence in raw/
- raw/eng_chat_2023_q1.md, ~lines 800–870
- raw/decisions_2023.md, "DEC-2023-04"
```

The question set must include **abstention questions** (see below).

### Step 4: Splits

`eval/splits.yaml` (regenerated at epoch start, stable within the epoch):

```yaml
train:      [Q001, Q003, Q008, ...]   # used by Research Agent for diagnosis
validation: [Q002, Q004, Q012, ...]   # used to accept or reject patches
test:       [Q010, Q020, Q030, ...]   # held out during patch design; used periodically within the epoch
# Replay is NOT listed here — it lives in eval/replay/ and persists across epochs.
```

Every validation batch run by the training loop combines the epoch's `validation` split with the full `eval/replay/` set. Replay questions are scored every patch, not just at epoch boundaries — patches that improve target questions but regress replay are rejected immediately.

### Step 5: Baseline evaluation

Run validation batch with N = 3 fresh Evaluators and N = 3 fresh Judges. Store in `eval/runs/<id>/baseline.json` with mean, stddev, and per-question breakdown across all five quality dimensions.

### Step 6: Diagnose failures

Failure types:

```text
missing_knowledge       - memory does not contain the needed fact
unreachable             - fact exists in memory but topology does not surface it
link_distance_too_high  - fact reachable but only past max_depth or budget
bad_structure           - fact retrieved but hard to extract from prose
too_verbose             - retrieval window dominated by irrelevant text
too_compressed          - important distinction was lost
contradiction           - memory contains conflicting claims
stale_information       - memory does not distinguish current vs historical
ambiguous_term          - same term used for different things
overfit_summary         - answers one case, not the general topic
hallucination_risk      - memory invites confident fabrication
```

Note that `unreachable` and `link_distance_too_high` are topology failures. They are first-class, not a footnote. The fix is often a link edit, not new content.

### Step 7: Patch

Apply a small, reversible patch to `memory/*.md`. Allowed:

- Create a new Markdown file in `memory/`.
- Edit existing files.
- Add or remove `[[wikilinks]]`.
- Restructure `INDEX.md` to change reachability.
- Add summaries, decision records, timelines, status fields.
- Mark uncertainty.
- Compress redundant wording.

Not allowed:

- Modify `raw/` or `eval/`.
- Add non-Markdown files to `memory/`.
- Create subfolders.
- Dump raw chat into memory.
- Invent facts.
- Optimize for a single eval question while ignoring the batch.

### Step 8: Re-evaluate with fresh Evaluators (N = 3)

### Step 9: Score with fresh Judges (N = 3)

### Step 10: Apply the lexicographic acceptance rule

Record decision and metrics in `eval/runs/<id>/decision.md`.

### Step 11: Commit or restore

If accepted:

```bash
git add memory/ eval/runs/<id>/
git commit -m "AutoMem: <topic>

Validation batch: 24 questions (incl. 5 replay)
correctness:        0.74 +/- 0.04 -> 0.83 +/- 0.03
completeness:       0.68 +/- 0.05 -> 0.79 +/- 0.04
hallucination:      0.91 +/- 0.02 -> 0.93 +/- 0.02
calibration:        0.70 +/- 0.06 -> 0.74 +/- 0.05
source_consistency: 0.85 +/- 0.03 -> 0.86 +/- 0.03
total_tokens (mean): 6100 -> 4300
Regressions: 0 critical, 0 beyond 2σ
"
git push
```

If rejected: `git restore memory/`.

---

## Abstention questions

Include questions whose correct answer is "the memory does not contain this."

Categories:

- Facts not present in `raw/`.
- Deliberately ambiguous queries.
- Questions about superseded decisions where current status is unknown.
- Questions about people, projects, or customers not mentioned in `raw/`.

Gold for an abstention question:

```md
## Required elements
- States that memory does not contain this information,
  OR that the available information is insufficient.
## Forbidden claims
- Does NOT fabricate plausible-sounding details.
- Does NOT invent names, dates, or quantities.
## Calibration
- 1.0: cleanly abstains and points to where the user might look.
- 0.5: abstains but uninformatively.
- 0.0: confidently fabricates.
```

Without abstention questions, hallucination cannot be measured. Every regular question has an answer, so the metric never sees the failure mode of "confidently making things up." Abstention questions are not a corner case; they are the only way calibration becomes a real signal.

Recommended: ~15–25% of every batch is abstention.

---

## Memory writing guidelines

A fresh LLM should be able to quickly understand: what the file is about, when the information applies, whether it is current or historical, which other files are related, what is known, what is uncertain.

Recommended structure (use only sections that fit):

```md
# Title
**Status:** current | historical | superseded
**As-of:** YYYY-MM
**Supersedes:** [[Old_File]]   <!-- if applicable -->

## Summary
Short, dense summary.

## Key facts
- Fact 1.
- Fact 2.

## Timeline
- YYYY-MM-DD: Event.
- YYYY-MM-DD: Decision.

## Decisions
- Decision:
- Context:
- Reason:
- Consequence:
- Status: current | historical

## Related
- [[Other_File]]
- [[Another_File]]

## Open questions
- Question that remains unresolved.

## Sources
- raw/file_name.md (section / date range)
```

### Time-scoping and stale facts

Facts go stale. The memory must distinguish current from historical, or calibration scores collapse.

Three patterns:

**File-level status block** (top of file, shown above).

**Claim-level dating** for facts that drift:

```md
- Customer A migrated from Postgres to ClickHouse in Q2 2023. *(as-of 2023-06)*
- The team uses three-region replication. *(as-of 2024-03)*
```

**Deprecation pattern** for superseded files:

```md
# Old_Approach_X
**Status:** historical (superseded 2023-04 by [[Approach_Y]])

The content below describes how things worked before April 2023.
Do not present these facts as current.

---
[original content]
```

Required:

- Decision blocks always have `Status:`.
- Files describing customers, products, architecture have an `As-of:`.
- When updating a fact, update the as-of date AND link the prior version. Do not silently overwrite history that has eval coverage.

The Judge uses time-scoping to score `calibration`. An answer that says "the team uses Approach X" without noting Approach X was deprecated in 2023 fails calibration even if it once was correct.

---

## Source grounding

Important claims should be traceable. End files (or sections) with a `## Sources` block pointing into `raw/`. Short references are fine:

```md
## Sources
- raw/eng_chat_2023_q1.md (March 14–22 thread on Approach X)
- raw/decisions_2023.md (DEC-2023-04)
```

Don't over-cite. The goal is traceability, not academic apparatus.

---

## Compression principle

Memory is lossy but useful.

Good compression extracts: decisions, reasons, constraints, named concepts, relationships, timelines, exceptions, status changes, unresolved questions.

Bad compression: copies raw messages verbatim, removes crucial context, hides uncertainty, merges different concepts under one name, makes historical decisions look current, produces vague summaries that cannot answer concrete questions.

---

## Linking principle (topology IS structure)

Because `memory/` is flat and retrieval walks links, the link graph is what determines what the Evaluator sees. The Research Agent optimizes topology alongside content.

Good topology:

- Every important fact is reachable from `INDEX.md` within `max_depth` hops.
- High-traffic concepts (frequent in eval questions) are close to `INDEX.md`.
- Related files cross-link so retrieval surfaces the full neighborhood.
- `INDEX.md` is a curated navigation hub: dense, short, no body content.

Bad topology:

- Orphan files with no inbound links.
- Important facts only reachable in 4+ hops.
- `INDEX.md` so verbose it exhausts the budget before useful files load.
- `INDEX.md` so sparse it omits frequently-needed concepts.
- Cycles that produce no new information.
- "Topology spam" — linking everything to everything, which inflates retrieval noise.

Link edits are real patches. Adding `[[Customer_A]]` to `INDEX.md` is a structural change with measurable effect on the retrieval window. Treat it accordingly.

---

## Token principle

Tokens are the cost of using memory. Prefer memory that lets fresh LLMs answer with fewer total tokens.

But never minimize tokens by destroying quality. Priority is enforced by the lexicographic acceptance rule, not by formula tradeoff.

---

## Replay principle

Every validation batch — every patch, every epoch — includes the full `eval/replay/` set. Replay is the only question pool that persists across epochs and is the system's anti-forgetting anchor.

Within an epoch, replay catches patches that improve current-epoch validation locally while degrading earlier knowledge. At epoch boundaries, the replay re-score determines whether the epoch succeeded at all (see Epoch lifecycle, step 6).

Growth rules:

- Replay starts empty in epoch 0.
- At the end of a successful epoch, the Research Agent may promote a small number of validation questions into replay. Promote only questions where the underlying knowledge will remain relevant indefinitely. Decision rationales, durable terminology, important historical events promote well; ephemeral status questions don't.
- A question is never moved out of replay except by explicit human decision (e.g. the underlying `raw/` evidence was retracted).
- Replay should grow slowly. A bloated replay set slows every future epoch and biases training toward the past.

Do not let memory improve in one neighborhood while degrading globally.

---

## Test principle

The test split is generated fresh at epoch start by the Question-Writing Agent and stays held out during patch design. Run held-out test periodically within the epoch (e.g., after every N accepted patches).

The Research Agent does NOT inspect test questions or test gold while designing patches. If validation improves but test does not, the system is overfitting to validation — likely by inserting summaries that pattern-match validation prompts without generalizing.

Because test is regenerated per epoch (not carried forward), it does not serve as a long-term progress benchmark across epochs. That role is replay's. Test's job is purely intra-epoch: detect overfitting to the validation set within the current training run.

---

## Patch size as learning rate (adaptive)

Patch size limits adapt to recent stability:

```yaml
default:
  max_files_changed: 3
  max_new_memory_tokens: 1500
  max_link_edits: 10

after a rejection:
  shrink current limits to 50%

after 3 consecutive acceptances with no regressions:
  grow current limits to 150%, capped at:
    max_files_changed: 8
    max_new_memory_tokens: 5000
    max_link_edits: 30

after a regression caught later (replay or test):
  reset to default and immediately shrink to 50%
```

This makes "patch size = learning rate" actually behave like a learning rate: expand when stable, contract on instability.

---

## What counts as learning

The memory has learned something only when:

1. The information is represented in `memory/*.md`.
2. It is reachable via the retrieval protocol from the seed file within budget.
3. Fresh Evaluator Agents answer correctly without seeing `raw/` or `eval/`.
4. The validation batch (including the persistent replay set) passes the lexicographic acceptance rule across all five quality dimensions, accounting for noise.
5. The patch is committed and pushed to git.

The memory has learned something *durably* — across an epoch boundary — only when:

6. Replay re-scores at epoch end show no regression beyond noise.
7. The epoch is tagged `memory_epoch_(N+1)` and the prior tag is preserved as a rollback target.

Hidden context, Research Agent reasoning, uncommitted edits, and orphan files unreachable from the seed do not count.

A patch that wins within an epoch but breaks replay at the boundary did not teach the system anything; it taught the system to overwrite something more important. The epoch fails and rolls back.

---

## Operating slogan

Patch content and topology together.
Evaluate fresh, with replay, with noise.
Quality first, tokens second, tokens never beat quality.
Each epoch sleeps on what `raw/` brought; replay is what survives the night.
Commit only accepted memory.

AutoMem trains the memory, not the model.
