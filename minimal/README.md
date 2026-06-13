# Minimal Scaffold

> The smallest thing you can build that *compounds*. An afternoon of work, copy-pasteable,
> no framework, no service. A directory of markdown notes + ~150 lines of Python.

If you only build one thing from this repo, build this. Everything in the root README is an
elaboration of the loop below. Start here, run it for a week, and the rest will make sense from
the inside.

The whole idea: **the model is stateless, so put the state in a vault the model reads and writes.**
Each session ends a little smarter than it started, because it left notes for the next one.

---

## What you need

- Any LLM you can call in a loop (API or local — anything that reads/writes text).
- Python 3 with its standard library. SQLite (built in) is the only "database."
- A git repo to hold the notes. That's the entire stack.

```
my-scaffold/
├── notes/                # the vault — one .md file per idea
├── index.py              # build/search a full-text index over notes/
├── loop.py               # the session loop: search → reason → write a note back
└── gate.py               # the commit gate: refuse broken notes
```

---

## 1. The note format

One file per idea. YAML frontmatter for structure, `[[wikilinks]]` for edges, prose for content.

```markdown
---
summary: One line describing this note (used for search + recall).
type: finding | decision | pattern | reference
status: current | superseded
tags: [retrieval, orchestration]
---

# Title

The actual content. Link to related notes inline like [[other-note-slug]] — the slug is the
filename without the `.md`. Links don't need to exist yet; an unresolved link just marks
something worth writing later.
```

Two rules make this a knowledge *graph* instead of a folder of files:
1. **Frontmatter is mandatory** — `summary` especially. The summary is what retrieval ranks on.
2. **Link liberally.** A `[[link]]` to a note you haven't written yet is a feature, not a bug.

---

## 2. The index — deterministic search (≈40 lines)

Don't use an LLM to *find* notes. Use SQLite's built-in full-text search (FTS5). It's instant and
it never hallucinates. Build the index once; rebuild it when notes change.

```python
# index.py
import sqlite3, sys, pathlib, re

DB = "vault.db"
NOTES = pathlib.Path("notes")

def build():
    con = sqlite3.connect(DB)
    con.execute("DROP TABLE IF EXISTS notes")
    con.execute("CREATE VIRTUAL TABLE notes USING fts5(slug, summary, body)")
    for md in NOTES.glob("*.md"):
        text = md.read_text(encoding="utf-8")
        # pull summary out of the frontmatter; body is everything else
        m = re.search(r"^summary:\s*(.+)$", text, re.MULTILINE)
        summary = m.group(1).strip() if m else ""
        con.execute("INSERT INTO notes VALUES (?,?,?)", (md.stem, summary, text))
    con.commit(); con.close()
    print(f"indexed {len(list(NOTES.glob('*.md')))} notes")

def search(query, k=5):
    con = sqlite3.connect(DB)
    rows = con.execute(
        "SELECT slug, summary FROM notes WHERE notes MATCH ? ORDER BY rank LIMIT ?",
        (query, k)).fetchall()
    con.close()
    return rows

if __name__ == "__main__":
    if sys.argv[1:] and sys.argv[1] == "search":
        for slug, summ in search(" ".join(sys.argv[2:])):
            print(f"  [{slug}] {summ}")
    else:
        build()
```

```bash
python3 index.py                            # build the index
python3 index.py search adverse selection   # query it
```

That's retrieval. When your vault grows past a few hundred notes and keyword search starts missing
conceptual matches, *that's* when you add a dense embedder (see the root README, layer 2) — not
before. Deterministic search first; learned search only when you can measure it helping.

---

## 3. The loop — search, reason, write back (≈50 lines)

This is the heart of it. Before the model reasons, it pulls relevant notes. After it reasons, it
writes a note back. That write-back is the entire compounding mechanism.

```python
# loop.py  — swap call_model() for your LLM client
import index, pathlib, re

NOTES = pathlib.Path("notes")

def call_model(prompt: str) -> str:
    """Replace with your LLM API/local call. Returns the model's text."""
    raise NotImplementedError

def slugify(title: str) -> str:
    return re.sub(r"[^a-z0-9]+", "-", title.lower()).strip("-")[:60]

def run(task: str):
    # 1. RETRIEVE — pull what the vault already knows about this task
    hits = index.search(task, k=5)
    context = "\n".join(f"- [[{slug}]]: {summ}" for slug, summ in hits)

    # 2. REASON — give the model the task + the retrieved context
    prompt = (
        f"Relevant prior notes from the vault:\n{context or '(none yet)'}\n\n"
        f"Task: {task}\n\n"
        "Answer the task. Then, on a final line starting with 'NOTE_TITLE:', give a short title "
        "for a note capturing the single most reusable thing you learned (or 'NOTE_TITLE: none')."
    )
    answer = call_model(prompt)
    print(answer)

    # 3. WRITE BACK — persist the lesson so the next session starts from here
    m = re.search(r"NOTE_TITLE:\s*(.+)", answer)
    title = (m.group(1).strip() if m else "none")
    if title.lower() != "none":
        slug = slugify(title)
        body = answer.split("NOTE_TITLE:")[0].strip()
        (NOTES / f"{slug}.md").write_text(
            f"---\nsummary: {title}\ntype: finding\nstatus: current\n---\n\n# {title}\n\n{body}\n",
            encoding="utf-8")
        print(f"\n→ wrote notes/{slug}.md")

if __name__ == "__main__":
    import sys
    run(" ".join(sys.argv[1:]))
    index.build()   # re-index so the new note is searchable next time
```

Run it twice on related tasks and watch the second run pull the first run's note into context.
That moment — the model citing something it learned an hour ago — is the whole point.

---

## 4. The gate — refuse broken notes (≈30 lines)

The one discipline that keeps a growing vault healthy: **never commit a note that links to nothing.**
A broken `[[wikilink]]` is debt; catch it at the commit boundary, not three months later.

```python
# gate.py — run before every commit; exit non-zero to block
import pathlib, re, sys

NOTES = pathlib.Path("notes")
slugs = {md.stem for md in NOTES.glob("*.md")}
broken = []
for md in NOTES.glob("*.md"):
    for link in re.findall(r"\[\[([^\]]+)\]\]", md.read_text(encoding="utf-8")):
        if link not in slugs:
            broken.append((md.stem, link))

if broken:
    for src, dst in broken:
        print(f"BROKEN LINK: {src} -> [[{dst}]] (no such note)")
    sys.exit(1)
print(f"gate OK — {len(slugs)} notes, 0 broken links")
```

Wire it as a git pre-commit hook (`.git/hooks/pre-commit` → `python3 gate.py`) and the vault can
never drift into an inconsistent state. This one invariant is the seed of the whole quality-gate
layer; add more invariants (frontmatter present, no orphans, provenance stamped) as you feel the
need for them — each one is just another check in this file.

---

## That's the whole compounding loop

```
   task ──▶ search vault ──▶ reason with context ──▶ write a note back ──▶ re-index
                ▲                                              │
                └──────────────────────────────────────────────┘
                         next task starts from a smarter vault
```

Four files, standard library only, no framework. Run it for a week on real work and you'll have a
vault that answers questions your first session couldn't — same model the whole time.

## Where to go next (in order of payoff)

1. **Dynamic retrieval everywhere** — make *every* substantive task start with a vault search, not
   just explicit "look this up" ones. This is the single highest-leverage habit.
2. **A second invariant in the gate** — e.g. "every note has a non-empty summary." Watch how one
   mechanical check changes behavior more than a paragraph of instructions ever does.
3. **Provenance** — stamp every auto-written note with who/why/when. The day you have two writers,
   you'll want to tell their notes apart.
4. **A dense embedder** — only once keyword search visibly misses things, and only with a held-out
   query set so you can prove the learned retriever actually beats FTS5. (Often it doesn't until the
   vault is large and the queries are conceptual.)
5. **Shaped workers** — when one model is doing everything, split off the cheap mechanical work to a
   cheaper model and keep the expensive model for synthesis + verification. See the root README,
   layer 3.

Each step is optional and independently useful. The four files above are the irreducible core —
build them first, live in them, and let the need for everything else announce itself.
