# 20 — Architectural decision records

Copy this file into the new repository as `docs/adr/README.md`, and § 4 as
`docs/adr/_template.md`.

---

## 1. What an ADR is for

An ADR is a short, dated note capturing **why** a structural choice was made, so a future
reader can re-evaluate it without re-deriving the context.

ADRs are not documentation. Documentation describes how the system **is**; an ADR
describes how the system **came to be that way** and what was traded to get there. If the
answer to "why is it like this?" requires reading commits, chat history, or somebody's
memory, write an ADR.

---

## 2. When to write one

Write one when **any** of these is true:

- The choice is hard to reverse — a data shape, a public contract, an infrastructure
  topology, a security boundary.
- The choice is non-obvious to a reader who only sees the final code, because the
  rejected alternative would also look reasonable.
- The same choice will be re-litigated in review unless the reasoning is written down.
- The constraint comes from outside the codebase — a legal requirement, a cost ceiling, a
  vendor limit, a lesson from an incident.

Do **not** write one for routine implementation, a behaviour-preserving refactor, or a
task note. Trivial choices belong in commit messages.

---

## 3. Process

1. Copy the template to `NNNN-short-kebab-title.md`. `NNNN` is the next zero-padded
   sequence number. **Never reuse a number.**
2. Fill it in. One screen is the target; two is the maximum.
3. Open the pull request that implements or revises the decision **with the ADR in it**.
   An ADR landing weeks after the code is half a record.
4. The reviewer treats the ADR as the load-bearing artifact. If the ADR is wrong or
   missing context, the review pauses on the ADR.

### 3.1 Status lifecycle

- **Proposed** — open for discussion. Code may exist but is not merged.
- **Accepted** — merged. In effect.
- **Superseded by NNNN** — replaced. Keep the file, add the link. Do not delete history.
- **Deprecated** — no longer the chosen path, not yet replaced. Note why.

**Editing an accepted ADR's decision is forbidden.** To change direction, write a new one
that supersedes it. Editing for typos, broken links, or clarity is fine — add a notes
section if the edit changes meaning.

### 3.2 Numbering, and the collision that keeps happening

Numbers are global, monotonic, and zero-padded to four digits.

**Check the number against every pushed branch, not only the default branch and not only
open pull requests.** A number free on the integration branch is not free if a pushed
branch has already claimed it — and that branch may have no pull request at all, so a
loop over open pull requests misses exactly the collision it exists to catch.

```
git fetch -q origin '+refs/heads/*:refs/remotes/origin/*'
for ref in $(git ls-remote --heads origin | awk '{print $2}' | sed 's#refs/heads/#origin/#'); do
  git ls-tree -r --name-only "$ref" -- docs/adr/
done | sort -u
```

**A commit message is not evidence of what a commit contains — read the tree.** A branch
that was renumbered keeps its original message, and citing that message reports a
collision that does not exist. Quote the file list, never the subject line.

### 3.3 The index is append-only

New rows go to the **end** of the index table, regardless of date. A merge conflict there
is therefore always "keep both rows", never "merge them".

More generally, for any append-only region — an index table, a registry array, the end of
a test file — **resolve the conflict by reconstruction:** discard the markers and
re-append both sides' additions in order. Editing between the markers to "reconcile" them
splices two independent additions into one broken thing.

---

## 4. The template

```markdown
# NNNN — <short title, present tense, no "we">

- **Status:** Proposed | Accepted | Superseded by [NNNN](NNNN-...md) | Deprecated
- **Date:** YYYY-MM-DD
- **Deciders:** <names or roles>
- **Tags:** <backend, security, mobile, data-model, ...>

## Context and problem statement

What was the situation that forced the decision? One short paragraph. Lead with the
problem, not the solution. If the reader does not understand the problem, the rest is
noise.

## Decision drivers

- <constraint or goal, one line each>
- <e.g. "must work offline">
- <e.g. "cannot break existing clients">

## Considered options

1. **Option A — <name>** — one-line summary
2. **Option B — <name>** — one-line summary
3. **Option C — <name>** — one-line summary

## Decision outcome

Chosen: **<option>**, because <one sentence — the dominant reason>.

### Consequences

- **Good:** <what this buys>
- **Bad:** <what this costs — be honest; a future reader will check>
- **Neutral but worth knowing:** <a dependency it locks in, a constraint it creates>

## Pros and cons of the options

*(Optional — include when the comparison is the load-bearing part.)*

## Links

- <the pull request that implements this>
- <related ADRs>
- <external reference>
```

---

## 5. The index

Keep a table at the bottom of the readme:

| # | Title | Status | Date |
|---|-------|--------|------|
| [0001](0001-....md) | ... | Accepted | YYYY-MM-DD |

---

## 6. The first ADRs of a new project

Write these before the second feature. Each one is a decision this specification makes,
and recording it in the project means a future reader can see it was chosen rather than
inherited:

1. The layered architecture and the dependency rule.
2. RPC contracts generated from declarations in the source.
3. The persistence choice, and why.
4. The tenancy model, including the primary-key rules.
5. The client applications and why they are separate packages.
6. The testing estate: what each suite is for and what gates a merge.
