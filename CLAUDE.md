# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in
this repository.

## Project shape

This is the **rules and toolset** project: where the Ico ruleset and the generic rules
compiler are written, branched, released and tagged. It is two git repositories inside
a third that holds neither of them:

```
rpg-master/              THIS directory — a repository too, tracking only the guidance
                         that spans both projects: this file, README.md, notes.md
  rpg-master/             REPO: the game engine + rules-toolset (the generic compiler)
    rules/demo/            the toolset's own test fixture — the only ruleset that
                           belongs to this repo
    rules-toolset/tools/   build.py, test_rules.py, rulesc.py, lint.py
  rules/
    ico/                  REPO: the Ico ruleset — rules/, book/, sim/, build/
```

The two inner repositories are ignored here rather than made submodules, and
`.gitignore` gives the reasoning at length: this is the working directory they are
*written* in, so a pin here would be a second thing to bump on every commit and would
record nothing they do not already record themselves.

Two things follow. A `git status` at the top level answers about the wrapper and its
three files, so a clean tree here says nothing whatever about the state of the rules or
the toolset — always `git -C` a specific repo, or `cd` into one. And the name
`rpg-master` appears twice, so paths written for another layout are wrong here and vice
versa; that has already produced documentation telling people to run commands that
could not work.

## Working across the projects

The adventures project (`ico-adventures`, elsewhere on this machine) holds both of
these repositories as git submodules and writes to them.
**[rpg-master/WORKING.md](rpg-master/WORKING.md)** covers how the two projects share
them. Read it before any task that involves the adventures project at all. What it
settles for sessions in *this* project:

- **This is the write side.** The clones here are where the rules and the toolset are
  developed, branched, released and tagged. The adventures project's submodule
  checkouts are read-mostly, and commit there only for a genuinely adventure-driven
  change such as a new creature.
- **Do not reach into `ico-adventures/` from a session here.** It holds a second
  working copy of both of these repositories, normally at older commits. One agent
  holding two paths to one repository at two commits can build one and test the
  other, or manufacture a conflict out of its own work. If a change needs to land
  there, it lands by push-and-pull through the remote, not by editing both copies.
- **The adventures project is a separate Claude project on purpose**, so that the
  ownership boundary below is crossed deliberately rather than in passing, and so
  that memory files stay unmixed — rules-prose conventions do not belong on scene
  text.
- **The exception is the seam**: the adventure compiler, `{% gm-only %}` generalising
  `{% book-only %}`, sharing `rulesc.py`'s parsing. Those are designed on both sides
  at once, so open one combined session deliberately and close it when the feature
  lands. Even then, edit the toolset here — the submodule copy consumes the result.

`WORKING.md` also steps out the procedures: extending the toolset, adding a creature
from the adventures side, bumping a pin, cutting a release, and resolving a `build/`
conflict. Follow them there rather than reconstructing them.

## Who may change what

- **[rules/ico/SHARING.md](rules/ico/SHARING.md)** — the ruleset's write surface.
  `rules/bestiary/` is open to any project, because writing an adventure creates
  monsters. Everything carrying a mechanic value is not: those numbers are measured
  by `sim/` against the whole system. The short form — *a change that moves no
  mechanic value may come from anywhere; a change that moves one comes from here.*
- **[rpg-master/SHARING.md](rpg-master/SHARING.md)** — the toolset. It knows about no
  particular game and must not learn. An addition has to be in the toolset's own
  vocabulary, usable by any ruleset, demonstrated in `rules/demo/`, and covered by
  `tools/test_rules.py` against both rulesets. The three output shapes are the
  interface, and changing one is agreed before it is written rather than reviewed
  after.
- **[rules/ico/VERSIONING.md](rules/ico/VERSIONING.md)** — what the three numbers
  mean and how a release is cut. The tiers describe what an adventure author must
  *do*: MAJOR a name went away, MINOR a name added or a value moved, PATCH no
  mechanic value changed.

Work on a feature branch rather than committing straight to `main` — that is how
changes get reviewed here.

## Commands

From `rpg-master/rules-toolset/`:

```bash
python3 tools/build.py ico          # rules/ico -> book.html, snippets.json, mechanics.json
python3 tools/test_rules.py ico     # 114 pipeline tests, exercising the real ruleset
python3 tools/build.py demo && python3 tools/test_rules.py    # the toolset's own fixture
```

Both tools also take `--path DIR` for a ruleset the name lookup cannot reach.

From `rules/ico/`, after building:

```bash
python3 sim/balance.py              # full balance report
python3 sim/balance.py --check      # gates only; exit 1 on failure
python3 sim/sweep.py -m <rule-id>.<mechanic_key>=<v1>,<v2>,...   # what-if a value
```

**`--check` does not currently pass**, and is not expected to: eleven gates fail and
each is an open tuning question `rules/ico/TODO.md` already carries. Run it either
side of a change to a rule value — the same eleven failures with the same numbers
means the change was neutral, a twelfth means it was not. The count tracks the rules
and has been six, then ten, then eleven; `rules/ico/sim/README.md` says why each time.

## The single-source rule

Rule documents (`rules/*.md` in any ruleset) are the golden source of both mechanics
and prose: mechanics in the frontmatter block, prose below. Prose never restates a
number — it interpolates it with `{{ mechanics.key }}`, or `{{ other-id:mechanics.key }}`
for another document's value. A literal number in prose matching one of that
document's own mechanics is a **hard build error**; the linter strips numbers inside
`` `inline code spans` `` first, so worked examples use those.

This is why `sim/*.py` never hardcodes a value either — every constant is read from
`build/mechanics.json`, and a missing key raises rather than defaulting.

`build/` is generated and tracked. Never hand-edit it; rebuild. On a merge conflict
there, take either side to clear the markers, rebuild, and commit the result — the
outputs are reproducible, so what matters is that the merged *sources* produced them.

Other conventions before editing a rule document:

- `[[other-id]]` links; `{% include other-id %}` splices a document into the book's
  include tree (positional, nests to any depth; a cycle is fatal, a diamond warns).
- `{% book-only %}...{% endbook-only %}` keeps content in `book.html` and strips it
  from `snippets.json` — for design notes and rationale, never for a mechanic value.
- Every Ico rule document ends with, in this order: the mechanic itself, a
  `## Example` using the recurring cast (Ashri, Dune, Sela, Bramm), and a
  `## Design note` wrapped in `{% book-only %}`.
- Document ids are one flat namespace across a ruleset and equal the filename stem, so
  a new file's name can collide with an existing document. The ids in use are the keys
  of `build/snippets.json`.
- Equipment is never banned by archetype or discipline. It interferes with skills and
  powers instead.

See [`rpg-master/rules-toolset/README.md`](rpg-master/rules-toolset/README.md) for the
full format reference.
