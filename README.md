# rpg-master

Working directory for the whole project: the game engine and client, the
placeholder 3D asset generator, and the rules pipeline — the toolset and
the rulesets it compiles.

```
rpg-master/                (this directory — its own git repository,
                            tracking only these notes and CLAUDE.md)
  rpg-master/               REPO: the game
    server.py, data.py       1. the engine — FastAPI + Socket.io
    public/                  the 3D client
    rpg-assets/              2. the placeholder model generator
    rules-toolset/           3. the rules compiler — generic, no game content
    rules/demo/              4. worked-example ruleset (exercises the toolset)
  rules/
    ico/                    REPO: 5. the Ico ruleset
```

**Naming note:** the outer working directory and the engine repository
are both called `rpg-master`, so `rpg-master/rpg-master/` is a real path
and not a typo. Paths written for one layout are wrong in the other,
which has already produced documentation telling people to run commands
that could not work. When following instructions from elsewhere, check
which of the two is meant.

## Three git repositories, two of them here

`rpg-master/` and `rules/ico/` are separate repositories with their own
histories, remotes and — for the ruleset — release tags. This outer
directory is a repository too, but only for what belongs to the working
directory itself: this README, `notes.md`, and `CLAUDE.md`. The two
nested repositories are gitignored rather than made submodules; the
reasoning is in `.gitignore`.

A third project, `ico-adventures`, lives elsewhere on this machine and
holds both of them as submodules. That means there are **two working
copies of each shared repository**, normally at different commits.

- **[CLAUDE.md](CLAUDE.md)** — how to work here, and the pointers below
  gathered in one place.
- **[rpg-master/WORKING.md](rpg-master/WORKING.md)** — how the two
  projects share these repositories: which working copy to open, why
  they are separate Claude projects by default, and step-by-step
  procedures for extending the toolset, adding a creature, bumping a
  pin, cutting a release, and resolving a `build/` conflict.
- **[rpg-master/SHARING.md](rpg-master/SHARING.md)** and
  **[rules/ico/SHARING.md](rules/ico/SHARING.md)** — who may write to
  what in each.
- **[rules/ico/VERSIONING.md](rules/ico/VERSIONING.md)** — what the
  version numbers mean. The ruleset is at 1.0.4; the toolset is
  unversioned, and a consumer's submodule pin is its version.

## The five pieces

### 1. `rpg-master/` — the game engine and 3D client

FastAPI + Socket.io server, Three.js client. Players and the DM connect
to the same page and get different tools by role. See
[`rpg-master/README.md`](rpg-master/README.md) for how to run it.

### 2. `rpg-master/rpg-assets/` — placeholder 3D models

The procedural low-poly character and prop generator (hero, goblin,
ogre, crate, rock) that the client currently renders. It sits inside the
engine directory because that is who consumes its output —
`rpg-master/public/assets/` holds copies of what it generates. See
[`rpg-master/rpg-assets/README.md`](rpg-master/rpg-assets/README.md).

### 3. `rpg-master/rules-toolset/` — the rules compiler

Turns a *ruleset* (a directory with `rules/*.md` and `book/*.md`) into a
long-form rulebook, in-game context-help snippets, and a pure-data file
the server can load its constants from. It knows nothing about `demo` or
`ico` specifically: which ruleset to build is an argument. See
[`rpg-master/rules-toolset/README.md`](rpg-master/rules-toolset/README.md)
for the full format reference — interpolation, links, includes, cycle
detection, the linter, and the honest limits.

### 4. `rpg-master/rules/demo/` — worked example

Six rules, three chapters, cross-links, mechanics interpolation: a
complete small ruleset that exercises every feature of the toolset. It
is the toolset's own integration test and the place a new feature is
demonstrated, which is why it ships with the toolset rather than
alongside the real ruleset.

### 5. `rules/ico/` — the Ico ruleset

A d20 system with no classes, built on a discipline-based
specialisation and advancement layer. Forty-four documents: the core
roll, the two hit point pools, character creation, combat and stances,
skills, disciplines, powers, equipment, and magic with a spell list of
forty-three spells. It also carries `sim/`, a balance simulator that
measures whether the resulting game is any good — a job the build
pipeline cannot do, since proving every number agrees says nothing
about whether the numbers are right.

Deliberately unfinished: the bestiary has one creature, ranged weapons
are unstatted, and around thirty skills have no governing attribute
yet. See [`rules/ico/README.md`](rules/ico/README.md) and its `TODO.md`.

## How the pieces relate, and do not yet

- **The toolset and the rulesets are decoupled by design.** The toolset
  takes a ruleset as an argument — `build.py demo`, `build.py ico`, or
  `--path` for one that lives anywhere else — and has no hardcoded
  knowledge of either. That is what makes `demo` and `ico` genuinely
  siblings rather than one being special-cased, and it is the property
  that has to survive every extension.
- **The engine still does not read either ruleset.** `data.py` holds
  its own constants (grid size, the dice notation regex) and nothing
  imports `rules_runtime.py`, which was written and tested to close
  exactly that loop. The open question is not whether but how: whether
  the server reads a ruleset's `build/mechanics.json` in place, or a
  deploy step copies it in. The ruleset now has real content and a
  version number, so the reason to defer the decision has gone.
- **The simulator cannot load a creature.** It measures the archetype
  panel against itself, so a bestiary entry's `challenge_level` is an
  author's estimate that nothing has checked. This is the gap that most
  affects the adventures project.
- **The asset generator and the rules pipeline are unrelated.** One
  produces 3D models, the other rules text and data; nothing connects
  them. Which model a creature renders as still lives in `data.py`'s
  `MODEL_KINDS` and the DM's spawn dropdown, not in any rule.

## Running things

Build and check the rules, from `rpg-master/rules-toolset/`:

```bash
python3 tools/build.py ico          # or demo
python3 tools/test_rules.py ico     # 114 pipeline tests
```

Measure the game, from `rules/ico/`, after building:

```bash
python3 sim/balance.py --check      # gates only; exit 1 on failure
```

The first two prove the book, the snippets and the server data agree on
every number. The third asks whether those numbers make a good game.
They catch entirely different things, and `--check` does not currently
pass — six gates fail, each an open tuning question `TODO.md` carries.

Run the game, from `rpg-master/`:

```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python server.py                    # → http://localhost:3000
```

Browse the compiled book and the snippet demo, from this directory:

```bash
python3 -m http.server 8000
# → localhost:8000/rules/ico/build/book.html
# → localhost:8000/rpg-master/rules-toolset/snippet-demo.html
```

Regenerate the 3D placeholders, from `rpg-master/rpg-assets/gen/`:

```bash
python3 build_assets.py
# then copy out/*.glb into ../../public/assets/models/ — see that
# project's README for the full loop, including validation and preview
```
