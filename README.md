# Scaffold Harness

> The infrastructure layer that makes a fixed LLM compound across sessions.
> Model-agnostic. Markdown-first. Built and run by one operator on one small machine.

Most "AI memory" stops at *retrieval* — stuff a vector DB, pull the top-k, paste it in. This repo
documents the layer underneath that: a **harness** that turns a stateless model into something that
accumulates — knowledge, quality infrastructure, and its own retrieval model — without ever changing
the model's weights.

The thesis in one line: **better harness × same model ≫ bigger model × no harness.** The model is a
swappable arm; the harness is the organism.

---

## The shape

Everything is plain markdown notes in a git repo (the **vault**) plus a set of small,
single-purpose scripts that read and write them. No framework, no service mesh — a directory of
notes and a handful of Python files an operator can read end-to-end.

```
                         ┌─────────────────────────────────────┐
   operator  ───────────▶│   orchestrator (the reasoning model) │◀──────── external world
 (outside the membrane)  └───────────────┬─────────────────────┘          (news, prices, papers)
                                          │  dispatches shaped work,
                                          │  synthesizes every return
              ┌───────────────┬───────────┼───────────┬────────────────┐
              ▼               ▼           ▼           ▼                ▼
        bounded-tool      one-shot     judgment+   live-world      autonomic
        multi-step        code/gen     full-tools  sensing         housekeeping
        ($0 arms)         ($0 arms)    (sub-tier)   (sub-tier)      (cron, $0)
              └───────────────┴───────────┬───────────┴────────────────┘
                                          ▼
                              ┌───────────────────────┐
                              │   VAULT (bloodstream)  │  every arm reads from + writes back to it
                              │   markdown + index +   │  → parallel work becomes *cumulative*,
                              │   graph + dense model  │    not just concurrent
                              └───────────────────────┘
```

Six layers, bottom to top. Each is independently useful; the compounding comes from the stack.

### 1. The vault — a markdown knowledge graph
Plain `.md` files with YAML frontmatter (`summary`, `type`, `status`, `domains`) and `[[wikilink]]`
edges. Typed edges (extends / validates / contradicts / caused-by / preceded-by) make it a real
graph, not just a folder. This is the substrate everything else reads and writes. Notes are the unit
of memory; the git history is the audit trail.

### 2. Retrieval — deterministic first, learned second
- **Construction–traversal separation**: the model *writes* notes (construction); deterministic
  tools *find* them (traversal). Using a model for both is how you get hallucinated citations and
  quadratic cost.
- **Lexical index** (FTS5 / BM25): millisecond keyword search over the whole vault.
- **Graph expansion**: seed on a query, expand 1–2 hops along wikilinks.
- **A vault-finetuned dense embedder**: a small sentence-transformer fine-tuned *on the vault's own
  notes* against held-out real queries. The repeated empirical finding here: **data moves beat
  architecture moves** — densifying training positives (more supervision) reliably beats clever
  re-ranking, multi-view indexing, or backbone swaps. Re-ranking a finetuned dense retriever's
  output mostly *adds noise*; the lift lives in the embedder.
- **Dynamic per-task retrieval** is the default: query the vault for *this* task, not a static
  context preload at session start.

### 3. Orchestration — the shaped-tendril model
One reasoning model is the start/end node: it dispatches work and synthesizes every return. The
workers ("tendrils") are **not interchangeable** — each is shaped to a task:

| shape | route |
|---|---|
| audit / classify · investigate + plan | a cheap bounded tool-loop agent (coordinator) |
| new-script write, structured generation | a cheap one-shot code model (office) |
| multi-file synthesis | bounded agent **+ a coverage receipt** (list every file read/skipped) |
| run-a-script-then-analyze | the orchestrator pre-runs it and passes output as *data* |
| judgment + full tool access | a higher-tier agent (real bash/file tools) |
| live-world / web / search | a sensing arm |
| autonomic housekeeping | cron-fired specialists |

Two rules that fall out of running this for real:
- **A termination signal is not a quality signal.** A worker reporting "done" means the loop ended,
  not that the work is true. The orchestrator's synthesis function *includes verification* — every
  return gets a correctness pass. The honest ceiling on parallelism isn't worker count; it's how
  many returns the orchestrator can actually verify.
- **Receipts make cheap workers fail loud instead of fabricating.** A spec that says "paste the
  command output before analyzing it" converts a confident-but-wrong answer into an honest blocker
  when the worker hits a wall. Fail-loud, never fail-empty.

### 4. Quality gates — invariants at the commit boundary
A pre-commit gate runs a set of typed invariants over the vault and code: no orphan notes, no broken
wikilinks, frontmatter schema valid, provenance present on background-writer rows, prose claims
scoped to a real data window, and so on. Anything that fails refuses the commit. New code passes a
three-tier check: compile → self-test on synthetic fixtures → smoke against *real* data (the third
catches the semantic/integration bugs the first two miss).

### 5. Self-improvement — the vault feeds its own model
Operational experience (reasoning traces, bug fixes, caught patterns) is captured as notes; a
periodic pass promotes stable patterns into rules, gated by grounded evidence so the system can't
promote itself into an echo chamber. Periodically the vault is mined into training pairs and the
dense retriever is re-fine-tuned on its own accumulated content — the knowledge graph literally
trains its own retrieval model, with a held-out eval set frozen *before* the mining to keep the
score honest.

### 6. Provenance & trust — membranes around untrusted input
Every background writer stamps `source / trigger / trust-tier`. Content authored by external models
or fetched from the web is treated as **untrusted-read** — quoted and framed, never executed as
instructions. Dispatches that include untrusted data wrap it in explicit tags so a downstream model
treats it as data, not commands. The operator is the one node *outside* the membrane — the
irreplaceable external error-signal that keeps the loop from going self-referential.

---

## Why it holds up

- **The model is droppable by design.** Any single model — frontier or cheap — can vanish overnight
  (deprecation, policy, rate-limit). Because arms are swappable and the vault persists, losing one
  model degrades a function by one voice; it never breaks the organism. This has been stress-tested
  in the wild.
- **Reproducible with any LLM that reads markdown.** Nothing here needs a specific provider. The
  templates and the minimal build live in `minimal/`.
- **One operator, one small machine.** This is not a lab system. It is a single person's setup,
  which is the point: the leverage is in the infrastructure design, not the compute.

## Start here

| If you are… | Read |
|---|---|
| Building your own scaffold | `minimal/` — the smallest version that compounds |
| Studying the architecture | `architecture/` — each layer in depth |
| Looking for the reusable ideas | `patterns/` — the design patterns, model-agnostic |

## Honest limitations

- Much of the "it works" evidence is single-operator and self-measured; treat headline numbers as
  directional, not peer-reviewed.
- The dense-retrieval finding ("data beats architecture") is specific to a private, finetuned-over-a-
  small-corpus setting — it may not transfer to generic large-corpus RAG.
- The orchestration quality rules are calibrated on a modest number of real runs; they're working
  priors, not laws.
- This documents *infrastructure methodology*. It is not a drop-in library; it's a blueprint plus
  the lessons from running it.

---

*Companion to a longer research write-up of the same system. This repo is the infrastructure /
harness view: how the substrate is built, not what any particular instance was used for.*
