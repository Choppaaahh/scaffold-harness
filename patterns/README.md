# Patterns

> The reusable ideas, abstracted from running the harness for real. Each one is model-agnostic and
> stack-agnostic — they're about *how the system is shaped*, not what it's built on. Most were
> learned by getting them wrong first.

Read these after `minimal/`. They're the difference between a scaffold that works for a week and one
that stays trustworthy as it grows.

---

## 1. Construction / traversal separation

**Problem.** It's tempting to let the model do everything — write the notes *and* find them ("read
my vault and tell me which note is relevant"). This is slow, expensive, and hallucinates citations
to notes that don't exist.

**Pattern.** Split the two roles. The **model constructs** (writes notes, synthesizes). **Deterministic
tools traverse** (find notes: full-text index, graph walk, embedding lookup). Never use the model
as the retrieval engine.

**Why it works.** Indexed retrieval is `O(log n)` and never invents a result; model-driven search
is `O(n)`, costs a forward pass per step, and will confidently cite `[[note-that-isnt-there]]`. The
model's judgment is the scarce resource — spend it on construction, not lookup.

**When not to.** Genuinely fuzzy "find me the vibe of X" queries where no index captures the intent —
but reach for that only after the deterministic path demonstrably misses.

---

## 2. Fail loud, never fail empty

**Problem.** A cheap worker (or any component) hits a wall it can't cross — a missing capability, a
file it can't read, a check that can't run. The dangerous failure mode is that it produces a
*confident, plausible, wrong* answer instead of admitting the wall.

**Pattern.** Force a **receipt**. Make the spec say: *"paste the command's raw output before you
analyze it"* or *"list every file you read and every file you skipped."* A component that must show
its evidence can't fabricate — when it hits the wall, the receipt step turns into an honest blocker
("could not run X because Y") instead of an invented result.

**Why it works.** Empirically observed twice in one day: the same task, run two ways. Without the
receipt, a model fabricated a result table it had no way to produce. With the receipt requirement,
a *different* model on the same wall wrote "COULD NOT RUN: <reason>" and stopped. The receipt is
cheap; the fabricated result is a landmine that detonates downstream.

**When not to.** Trivial deterministic steps where there's nothing to fabricate (a compile check
either passes or it doesn't).

---

## 3. A "done" signal is not a quality signal

**Problem.** A worker reports completion. The orchestrator treats "done" as "correct" and moves on.

**Pattern.** **Termination ≠ truth.** A done-sentinel certifies that the loop *ended* — not that the
output is right. The orchestrator's job includes a verification pass on every return, not just
aggregation. The honest ceiling on how much you can parallelize isn't worker count; it's how many
returns the orchestrator can actually *check* per unit time.

**Why it works.** When several cheap workers were run in parallel on real tasks, *every one*
reported "completed" — and one had fabricated, one had silently covered a third of its scope. The
only thing that caught it was the orchestrator re-checking each return against reality. Build the
verification in; don't let "done" be load-bearing.

**When not to.** Pure mechanical transforms with an objective pass/fail already attached (tests,
compilers) — the check is the signal.

---

## 4. Droppable arms

**Problem.** You wire your whole system around one model (the best one, the cheapest one, doesn't
matter). Then it gets deprecated, rate-limited, price-changed, or — this happened — pulled by a
regulator overnight. Your organism breaks.

**Pattern.** Treat **every external model as a droppable arm.** The persistent thing is the vault and
the orchestration; the models are interchangeable workers plugged into shaped slots. Losing one
degrades a function by one voice — it never breaks the system. Keep a fallback chain; make swapping a
model a one-line change, not a re-architecture.

**Why it works.** Stress-tested in the wild: a flagship model was removed from availability with no
notice. The cost to a droppable-arms system was ~zero — the affected role just ran with one fewer
option. A system with a load-bearing single model would have been down.

**When not to.** Never — but be honest that *some* capabilities (a specific fine-tune you trained,
your own vault) genuinely aren't swappable. Those you own; external models you rent.

---

## 5. Data beats architecture (for retrieval on a private corpus)

**Problem.** Retrieval is missing things. The instinct is to add machinery: a re-ranker, multi-vector
indexing, a fancier model.

**Pattern.** On a small, private, fine-tuned corpus, **densifying the training data beats adding
architecture** almost every time. More/better supervision for the embedder reliably beat re-ranking
its output, multi-view indexing, and backbone swaps. Re-ranking a *fine-tuned* dense retriever's
output often just *adds noise* — the retriever is already calibrated; the extra stage injects
off-distribution signal.

**Why it works.** A fine-tuned embedder has already learned your corpus's geometry. A re-ranker
trained on someone else's distribution is, relative to that, a worse judge. The lift lives in the
embedder's training data, not in post-processing its results.

**When not to.** Large generic corpora with off-the-shelf embeddings — there, re-ranking genuinely
helps, because the first-stage retriever *isn't* calibrated to your data. Measure on a held-out set
before you trust either way.

---

## 6. Discipline at the layer, not instruction in the prompt

**Problem.** You write a rule: "always search the vault first," "never commit a broken link,"
"remember to add provenance." It works for a day, then quietly stops happening under load.

**Pattern.** Convert the instruction into a **mechanical check at the layer where the action happens.**
"Never commit a broken link" becomes a pre-commit hook that *refuses* the commit. "Always add
provenance" becomes a writer function that *can't* emit a row without it. The discipline stops being
something to remember and becomes something the system enforces.

**Why it works.** A remembered rule degrades with cognitive load; a mechanical gate doesn't get
tired. One real invariant changes behavior more than a paragraph of "please remember to…" ever does.
The text version is a wish; the gate is a fact.

**When not to.** Genuinely judgment-laden calls that can't be mechanized — there, the text rule plus
an outside reviewer is the best you can do. Don't pretend a fuzzy thing is a crisp check.

---

## 7. The operator is outside the membrane

**Problem.** A self-improving system that only ever reads its own output drifts. It measures itself
with itself, promotes its own patterns on its own say-so, and slowly optimizes for looking good to
itself — an echo chamber with no external ground truth.

**Pattern.** Keep a node **outside the membrane** — a human (or a genuinely decorrelated reviewer)
whose error-signal the system cannot generate from within. Gate self-promotion on *grounded*
evidence (something measured against the outside world), not internal coherence. Treat anything the
system or an external model wrote as **untrusted-read** — quote and frame it, never execute its
instructions as your own.

**Why it works.** The single most valuable correction in a long session is reliably the one the
system *couldn't have produced itself* — the outside catch. Capability-from-inside doesn't substitute
for a position outside the frame. Buy outside positions deliberately; they're the thing that keeps the
loop honest.

**When not to.** This one has no off switch. The day you stop having an outside check is the day the
drift becomes invisible to you.

---

## How they fit together

The four `minimal/` files are the *mechanism*; these seven patterns are the *discipline* that keeps
the mechanism trustworthy as it scales:

- **1 & 5** keep retrieval honest and cheap.
- **2 & 3** keep cheap/parallel workers from lying to you.
- **4** keeps you robust to losing any single model.
- **6** keeps rules from rotting into wishes.
- **7** keeps the whole self-improving loop from becoming a hall of mirrors.

None of them need a specific model, framework, or domain. They're what's left when you strip the
implementation away — the shape of a scaffold that stays trustworthy.
