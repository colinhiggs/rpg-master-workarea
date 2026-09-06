# rpg-master

Working directory for the whole project: the game engine/client, the
placeholder 3D asset generator, and the rules pipeline (toolset + the
rulesets it compiles). Four pieces, laid out as you asked:

```
rpg-master/                (this directory)
  rpg-master/               1. the game — server + 3D client
    rpg-assets/               nested: the placeholder model generator
  rules-toolset/            2. the rules compiler — generic, no game content
  rules/
    demo/                   3. worked-example ruleset (exercises the toolset)
    ico/                    4. your ruleset — empty, ready to write into
```

**Naming note, flagged rather than silently "fixed":** the outer
working directory and the engine subdirectory are both named
`rpg-master`, per your spec (§1 called the engine's subdirectory
`rpg-master`, "a working title," while the enclosing directory is also
`rpg-master`). I built it exactly as literally specified — `rpg-master/
rpg-master/` — rather than guess you meant something else. If that
nesting was a copy-paste accident and you'd rather the engine live in,
say, `rpg-master/engine/`, that's a rename away (update this README's
tree and the paths below to match).

## The four pieces

### 1. `rpg-master/` — the game engine and 3D client

FastAPI + Socket.io server, Three.js client. Players and the DM connect
to the same page and get different tools based on role. See
[`rpg-master/README.md`](rpg-master/README.md) for how to run it.

### `rpg-master/rpg-assets/` — placeholder 3D models

The procedural low-poly character/prop generator (hero, goblin, ogre,
crate, rock) that the engine's client currently renders. Nested inside
the engine directory because that's who consumes its output —
`rpg-master/public/assets/` holds copies of what this generates. See
[`rpg-master/rpg-assets/README.md`](rpg-master/rpg-assets/README.md).

### 2. `rules-toolset/` — the rules compiler

Turns a *ruleset* (a directory with `rules/*.md` + `book/*.md`) into a
long-form rulebook, in-game context-help snippets, and a pure-data file
the server can load its constants from. Knows nothing about "demo" or
"ico" specifically — which ruleset to build is a command-line argument.
See [`rules-toolset/README.md`](rules-toolset/README.md) for the full
format reference (interpolation, links, includes, cycle detection, the
linter, honest limits).

### 3. `rules/demo/` — worked example

Six rules, three chapters, cross-links, mechanics interpolation — a
complete small ruleset that exercises every feature of the toolset.
Useful as a template and as the toolset's own integration test; not
meant to be your actual game's rules.

### 4. `rules/ico/` — your ruleset

Empty, waiting for content. I added one file —
[`rules/ico/README.md`](rules/ico/README.md) — with the minimum shape
needed to get a first build working (a `rulebook.md` and one example
rule), since an entirely empty directory gives `build.py` nothing to
point at and no way to show you what "done" looks like. No actual game
content is in there; delete the README if you'd rather start from
nothing.

## How the pieces relate (and don't, yet)

- **`rules-toolset/` and `rules/*` are decoupled by design.** The
  toolset takes a ruleset path as an argument (`build.py demo`,
  `build.py ico`, or `--path` for anything elsewhere); it has no
  hardcoded knowledge of either ruleset. That's what makes `demo` and
  `ico` genuinely siblings rather than one being special-cased.
- **The engine does NOT yet read from either ruleset.** `rpg-master/
  data.py` still hardcodes its own game constants (starting HP, grid
  size, the dice regex). `rules-toolset/tools/rules_runtime.py` is
  written and tested to close that loop, but wiring it in is a
  decision I flagged last time and haven't made unilaterally: whether
  the server reads a ruleset's `build/mechanics.json` directly, or a
  build/deploy step copies it into the server's own directory. Whoever
  writes `ico`'s actual rules, that's the moment to make that call —
  no point wiring the server to `mechanics.json` before `ico` has any.
- **`rpg-assets` and the compiled rulebook are unrelated pipelines**
  that happen to both feed the same game: one produces 3D models, the
  other produces rules text and data. Nothing currently connects them
  (e.g. no rule like "goblins render as the goblin model" is derived
  from anywhere) — that mapping still lives in `data.py`'s
  `MODEL_KINDS` and the DM's spawn dropdown.

## Running things, in order

```bash
# 1. Build the demo ruleset (or ico, once it has content)
cd rules-toolset
python3 tools/build.py demo
python3 tools/test_rules.py

# 2. Run the game
cd ../rpg-master
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python server.py
# → http://localhost:3000

# 3. Browse the compiled rules / try the in-game-help demo
cd ..   # back to rpg-master/ (the outer one)
python3 -m http.server 8000
# → localhost:8000/rules/demo/build/book.html
# → localhost:8000/rules-toolset/snippet-demo.html

# 4. Regenerate 3D placeholder assets, if you change them
cd rpg-master/rpg-assets/gen
python3 build_assets.py
# then copy out/*.glb into ../../public/assets/models/ (see that
# project's README for the full loop, including validation/preview)
```
