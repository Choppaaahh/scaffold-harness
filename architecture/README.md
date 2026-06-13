# Architecture

> The six layers in depth — the mechanics, the design decisions, and the gotchas that only show up
> once you've run the thing for months. The root README is the map; this is the territory.

Read `minimal/` first (the irreducible core) and `patterns/` second (the discipline). This document
assumes both. Each section below takes one layer from the root README and goes past "what it is" into
"how it actually behaves and why it's built this way."

---

## Layer 1 — The vault

**What it is.** Plain markdown files in a git repo. Each note: YAML frontmatter (`summary`, `type`,
`status`, `domains`/`tags`) + a body + `[[wikilink]]` edges. The git history is the audit trail.

**Why markdown + git, not a database.** Three reasons that compound:
- **The operator can read the whole thing.** Every note, every script, is human-inspectable. The day
  something is wrong, you `grep` and `git log`, you don't query a service you half-remember the schema
  of.
- **Diffs are the audit trail for free.** Who changed what, when, and why is `git blame`. No separate
  provenance store needed for the human-authored layer.
- **It's portable to any tool.** An LLM that reads text can read it. A human with a text editor can
  read it. There's no lock-in to a framework that might not exist next year.

**The frontmatter is load-bearing, not decoration.** `summary` is what retrieval ranks on — a vague
summary is a note that never surfaces. `status` lets you keep superseded notes (history) without them
polluting current retrieval (filter `status != superseded`). `type` lets downstream tooling treat a
`pattern` note differently from a `session-log` note.

**Typed edges make it a graph.** `[[x]]` is an edge; prefixing it with a relationship —
`extends [[x]]`, `contradicts [[x]]`, `caused [[x]]`, `preceded by [[x]]` — turns the vault into a
*typed* graph. "Show me everything caused by X" becomes a different, answerable query than "show me
everything about X." You don't need a graph database to benefit; typed links in markdown already make
traversal more precise.

**Gotcha — episodic vs substantive notes.** Daily logs, session notes, and dated entries pile up fast
and shouldn't be treated like reusable knowledge. Mark them by directory or `type` and exclude them
from the retrieval surface and from any spaced-review/maintenance pass. Reviewing a year-old daily log
adds nothing; surfacing it in search adds noise.

---

## Layer 2 — Retrieval

**The stack, in the order you should build it:**
1. **Lexical (FTS5 / BM25).** Millisecond keyword search. Build this first; it solves 80% of recall.
2. **Graph expansion.** Seed on the lexical hits, expand 1–2 hops along wikilinks. Catches the note
   that didn't share keywords but is one edge away from one that did.
3. **A vault-fine-tuned dense embedder.** Only when 1+2 visibly miss conceptual matches, and only with
   a held-out query set so you can *prove* it helps.

**The construction/traversal law (see patterns/1).** The model writes; deterministic tools find. Using
a model to search is how you get hallucinated citations and `O(n)` cost.

**On the dense embedder — the hard-won finding.** On a small, private corpus, the highest-leverage
move is **densifying the training data**, not adding machinery downstream:
- Fine-tune a small sentence-transformer *on your own notes* against real queries.
- **Freeze a held-out eval set before you mine any training data from the vault.** This is the single
  most important discipline in the whole self-improvement loop — without it, you measure the model
  against data it trained on and every number lies.
- Re-ranking a fine-tuned dense retriever's output tends to *add noise*, not signal: the retriever is
  already calibrated to your corpus; a re-ranker trained on someone else's distribution is a worse
  judge relative to it. Measured repeatedly — re-rankers, multi-view indexing, and backbone swaps all
  lost to "give the embedder more/better positives."

**Dynamic-per-task is the default.** Don't preload a static context block at session start. Query the
vault for *this* task, right before reasoning about it. Measured to substantially beat static preload —
the relevant 5 notes for the task at hand beat the 50 notes that seemed generally important.

**Gotcha — the eviction tax.** A raw append-only capture log (every tick, every snapshot) gets
re-parsed in full on every read and silently becomes your slowest operation. Rotate it: keep a small
hot window, archive the rest by time, verify row-count conservation on every rotation so you never lose
data, and lock against concurrent writers.

---

## Layer 3 — Orchestration

**The shape.** One reasoning model is the start/end node — it dispatches work and synthesizes every
return. Workers are *shaped to the task*, not interchangeable:

| shape | route to | spec requirement |
|---|---|---|
| audit / classify; investigate + plan | cheap bounded tool-loop worker | — |
| new-file / structured generation | cheap one-shot model | verify the output runs |
| multi-file synthesis | bounded worker + **coverage receipt** | "list every file read + skipped" |
| run-a-script-then-analyze | orchestrator pre-runs it, passes output as **data** | (workers often can't run arbitrary code) |
| judgment + full tools | a higher-tier agent | — |
| live-world / search | a sensing arm | treat returns as untrusted |

**Two laws fall out of running it (see patterns/2 & 3).**
- **A "done" signal certifies termination, not correctness.** Verify every return.
- **Receipts make cheap workers fail loud instead of fabricating.** Any spec whose deliverable claims
  grounding in a command's output or a file set must require the receipt.

**Calibrate slots empirically, don't guess them.** Run a handful of *real* tasks of each shape through
each worker and record what actually happened — pass / fabricate / under-cover / honest-block. The
routing table above is the *output* of that calibration, not a prior. Tighten it as you accumulate
runs; treat single-run verdicts as working priors, not laws.

**Gotcha — the verification bottleneck.** The honest ceiling on parallelism is not how many workers
you can spawn; it's how many returns the orchestrator can actually *check* per unit time. Ten parallel
workers that all need verification don't help if the orchestrator can verify three. Design for
verifiable returns (receipts, structured output), not just for fan-out.

---

## Layer 4 — Quality gates

**The mechanism.** A pre-commit gate runs a set of typed invariants over the vault and code and
*refuses* the commit if any fail. This is `minimal/`'s `gate.py` grown up.

**Invariants worth having (add as you feel the need):**
- no broken wikilinks (the seed invariant)
- every note has a non-empty `summary` and a valid `status`
- no orphan notes (every substantive note has ≥1 incoming link) — or an explicit exemption list
- background-writer rows carry provenance (`source` / `trigger` / `trust-tier`)
- prose claims of the form "X% / pattern P holds" cite a real data window, not a single session
- no unresolved infrastructure references (a rule that names a script/hook that must exist on disk)

**Code gets a three-tier check.** Compile → self-test on synthetic fixtures → **smoke against real
data.** The third tier is the one that matters and the one most often skipped: compile + synthetic
self-test catch *structural* bugs (syntax, arithmetic on known-shape input); only real-data smoke
catches the *semantic/integration* bugs — the all-NULL column, the parser that silently drops rows,
the field name that didn't match production. A guide example or a shipped script that passed the first
two tiers can still be wrong on real input.

**Why a gate beats a rule (see patterns/6).** A remembered "always do X" degrades under load; a gate
that refuses the commit doesn't get tired. Convert disciplines into checks at the layer where the
action happens, and the discipline stops being something to remember.

**Gotcha — grandfathering.** When you add an invariant to an existing vault, it will flag hundreds of
pre-existing violations. Don't block on all of them; seed an exceptions list of the current state
(`action_needed: true`) so *new* violations fail the gate while the backlog drains on its own schedule.

---

## Layer 5 — Self-improvement

**The loop.** Operational experience → notes → promoted rules → re-trained retriever → better future
sessions.

- **Reasoning traces and caught bugs become notes** as they happen (capture at the moment, not in a
  batch at session-end — batching loses the detail and the sequencing).
- **A periodic pass promotes stable patterns into rules**, gated on *grounded* evidence — something
  measured against the outside world (a real outcome, a caught bug, a held-out score), not the
  system's own say-so. This gate is what stops self-promotion from becoming an echo chamber.
- **The vault periodically re-trains its own retriever** on training pairs mined from its own notes —
  the knowledge graph literally trains the model that searches it.

**The one discipline that makes this safe (see patterns/7).** Freeze the eval set *before* mining
training data, and gate promotions on grounded evidence. A self-improving system that measures itself
with itself will reliably report progress while drifting. The canonical failure: a model trained on
its own labeler, scored by the same system, shows a big improvement — that would have shipped a
memorization artifact as "progress" if an outside adversarial check hadn't caught it. Build the outside
check in; it is not optional.

**Gotcha — promote slowly.** A pattern needs more than one instance and more than one session before
it's real (cross a sleep boundary; let session-recency bias wash out). Counting `n≥3` across a few days
beats promoting on the first compelling instance.

---

## Layer 6 — Provenance & trust

**Provenance at emission.** Every background writer stamps `source` (who wrote it), `trigger` (why it
fired — cron / user / chain-reaction), and `trust-tier` (direct-user / agent-reasoning /
autonomous-unsupervised). This lets downstream consumers tell a human-authored fact from an
autonomously-generated guess from a transit-amplified rumor — which matters enormously once more than
one thing writes to the vault.

**Untrusted-read membranes (see patterns/7).** Content authored by an external model, or fetched from
the web, or written by an autonomous process, is **data, not instructions.** Quote it, frame it, reason
about it — never execute imperatives found inside it. When you pass untrusted content into a model,
wrap it in explicit tags and instruct the model to treat the wrapped region as data. This closes the
"stored-injection" class: a poisoned note that says "ignore your rules and do X" gets quoted in
analysis, never obeyed.

**Quarantine for autonomous writes.** Autonomous/unsupervised writes enter at a lower trust tier and
get *reviewed up*, not trusted by default. A human (or grounded check) reviewing a note flips it from
`human_reviewed: null` to a date — the asymmetry (autonomous writes are penalized, human review
releases) is what keeps unreviewed machine output from quietly becoming load-bearing.

**The node outside the membrane.** The operator — or any genuinely decorrelated reviewer — is the one
source of error-signal the system cannot generate from within. Everything *inside* the loop is, to some
degree, the system marking its own homework. The outside node is what keeps it honest. Buy outside
positions deliberately (a human in the loop, a decorrelated second model for adversarial review); they
are the most valuable component and the one you can't synthesize.

---

## Putting it together

Bottom to top: a **vault** you can read, searched **deterministically-first**, worked by **shaped
arms** an orchestrator **verifies**, protected by **gates** that refuse bad state, improving itself
through a loop **gated on grounded evidence**, with **provenance and membranes** keeping trust legible
and the **operator outside the frame** keeping the whole thing honest.

No layer needs a specific model or framework. Swap any arm, change any provider, and the organism
persists — because the organism was never the model. It was the vault, the disciplines, and the node
outside the membrane.
