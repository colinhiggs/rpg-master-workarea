# On the possibility rulesets with logic

Before answering this I want to look at what the server actually needs from a ruleset — the interface surface is the whole question.

Read the server, and it reframes the question usefully.

## What's actually there

`data.py` has almost no game logic. It tracks tokens with `x`, `y`, `hp`, `maxHp`, a turn-order list, and a generic dice roller. There is no attack, no damage, no armour, no skills. **rpg-master today is a virtual tabletop, not a rules engine** — humans adjudicate, the server remembers.

That matters, because the first thing that breaks for Ico isn't the attack formula. It's that a token has one `hp` and Ico has *two* pools plus stamina and spirit. The shape of state blocks you before any formula does.

## The ladder

Pluggability isn't one thing. It's four rungs, and they cost very differently:

| Rung | What varies | Ico example | Cost |
|---|---|---|---|
| **0. Values** | Numbers | `dodge_targeting_base: 10` | **Done.** `mechanics.json` + `rules_runtime.py` |
| **1. State shape** | What a creature *has* | core + mastery HP, stamina, spirit | Moderate — touches data, persistence, UI |
| **2. Formulas** | How a number is computed | `weapon + margin/2 + skill/8` | Low, once rung 1 exists |
| **3. Pipeline shape** | Which *steps* exist | margin→damage vs separate damage roll | Low **if** logic is code; very high if it's a DSL |
| **4. Arbitrary conditionals** | "unless the target is undead" | — | Unbounded |

Your example — margin-derived damage versus a separate damage roll — is rung 3. But rung 1 is what's blocking you *now*.

## My recommendation: don't build a DSL

A declarative language powerful enough for rung 3 is a programming language, and you'd be writing an interpreter, error messages and a debugger for an audience of one. Instead: **a ruleset ships a Python module implementing a small interface, and `mechanics.json` stays the data half.**

```python
class Ruleset:
    def pools(self)     -> list[Pool]        # what a creature has
    def actions(self)   -> list[ActionSpec]  # what the UI should offer
    def resolve(self, action, actor, target, params) -> Outcome
```

The load-bearing piece is `Outcome`: **a list of opaque state deltas plus log lines.** The server applies deltas and broadcasts them without ever learning what an attack is. That's what keeps `data.py` generic across systems whose pipelines don't resemble each other.

**You already have the prototype.** `sim/model.py` is Ico's resolution logic reading nothing but `mechanics.json`, fail-fast on missing keys, ~600 lines, and continuously exercised by the balance gates. That is a ruleset logic module in all but name — it just currently reports to a simulator instead of a server. Point both at one module and the game's rules are validated by the balance harness on every run, which is a much stronger guarantee than most systems get.

## The honest cost

**Executable logic breaks the single-source guarantee.** Right now the linter proves prose and data agree. It cannot prove that `resolve()` implements what the book says — code can silently contradict the rules. Sharing the module with the simulator mitigates this (the gates assert behaviour against `mechanics.json`), but it doesn't eliminate it. And if rulesets ever become shareable, "drop in a folder and it executes Python" is a security question you'd have to answer.

**The expensive part is the client, not the rules.** A single HP bar becomes N ruleset-declared pools; fixed buttons become ruleset-declared actions; Ico specifically needs a "declare your difficulty" control that no other system wants. Rungs 0–2 are a few days of server work. The UI that follows is the real project.

## What house rules actually need

Worth separating out, because it's the reassuring half: **most house rules are rung 0 and work today.** "Crits do triple", "full plate is 9 not 8", "you get 20 starting mastery points" are all `mechanics.json` edits — rebuild, and the book, snippets and server all move together. A house rule needing rung 3 isn't a house rule, it's a different game.

## The first step I'd take

Not the plugin interface — rung 1, narrowly:

Replace `token.hp`/`maxHp` with a ruleset-declared set of named pools, with the current single-pool behaviour as what the demo ruleset declares. That forces the state-shape question while the codebase is still 400 lines, proves the concept against Ico's two pools, and is useful on its own whether or not you ever go further up the ladder.

I'd hold off until ico is functional, as you said — but that one change is worth making *before* the client grows, because retrofitting it later means touching every piece of UI twice.

# Handoff, 8 September 2026

Written because the work is moving to another laptop, and a session transcript is a
poor place to keep the only copy of where things stand.

## Where the three repositories are

`rules/ico` sits at `72b9be0`, two commits past `v2.2.0`, level with its remote.
`rpg-master/` (the toolset) sits at `250ff1d`, also level with its remote. Both can be
re-cloned from GitHub and lose nothing.

This wrapper repository now has a remote of its own,
`github.com/colinhiggs/rpg-master-workarea`, which is what makes the three of them
movable together. It did not have one when this note was first written, and the
difference matters: without it, these three files and their history lived in a single
`.git` directory on a single machine, and copying the tree with `.git` included was the
only way to keep them.

What has not changed is that the inner repositories are ignored here rather than made
submodules, and `.gitignore` gives the reasoning at length. So a clone of the work area
gives you this file, `CLAUDE.md` and `README.md` and nothing else; `rpg-master/` and
`rules/ico/` are cloned into place separately from their own remotes. Three clones, not
one recursive one.

## What the last stretch of work did

Three things, in order.

**The simulator got fast.** `sim/balance.py --check` went from about thirteen minutes
to about two, across three commits, with the output byte-identical at every step —
verified by dumping duel, skirmish, contribution and offence figures on a fixed seed
and diffing. The wins were a mechanics lookup cache invalidated by generation,
a damage curve computed once per pairing instead of once per margin, `copy.copy` in
place of `deepcopy` on the per-trial character refresh, derived caches for faces,
conditions and spell choice, and sharing one planning pass between `_plan` and
`expected_offence`. `sim/README.md` states the caching contract: anything derived from
mechanics hangs off `M.derived` and dies with `M.invalidate()`. A pre-existing bug fell
out of that — two module-level caches were keyed without the mechanics they derived
from, so `sweep.py` on those keys had been measuring the value it had just replaced.

**Positions became real.** Distance is a single integer, because Ico's diagonals cost
one square. Duels open at the hero's own reach and the arena matches it; crowds advance
against a finite retreat budget. That last part was a real defect on the first pass —
capping the gap at the arena does not bind, because a hero only has to restore the
opening gap and can then retreat forever. The machinery was inert until the four area
spell families were given a range, which was a gap in the rules rather than a fault in
the model; with a range they clear crowds untouched rather than merely faster.

**Armour stopped being free for casters.** The skill penalty named only its dodge
consequence, which is why every caster in the sim bought full plate. It now applies to
spellcasting as well, scoped there rather than to everything drawing on spirit, so an
armoured commander's Rally is untouched. Alongside it, `choose_gear` was maximising the
best *single* action, which priced a hybrid's second capability at zero and sold the
spellblade a suit that switched off half of it; the fix is a legality filter
(`casting_survives_the_kit`) rather than an invented weighting. Nothing is forbidden
anybody — the kit interferes, and a build declines to buy what would cripple it.

## The decision that is open

`--check` fails fourteen gates. That number is not a target and it moves: it has been
six, ten, eleven and now fourteen, and `sim/README.md` says why each time. Read the
numbers rather than the count — several failures sit within a hundredth of a bound, and
one seeded stream feeds every duel, so an early change re-rolls every later pairing.

The substantive question is the caster gap, and `rules/ico/TODO.md` carries it in full.
It decomposes as roughly 1.99x on offence and control multiplied by 1.68x on survival. Guards
cannot close it: `MAX_SELF_GUARD_RATIO` is a threshold and not a knob, and crediting a
guard at full value still leaves the evoker at 164 against a required 197. On the
damage side, `damage_per_casting_skill_step: 3 -> 1` takes the gates from fourteen to
six, but it overpays — casting skill is already at per-blow parity at step 3. So the
decision is not which number to pick. It is whether a caster's survival deficit ought
to be repaid in damage. If it should, step `1` is the change and the design note in
`damage.md` has to say why the rate is steeper than parity. If it should not, the
missing piece is defensive, the guards cannot supply it, and it would have to be
something new.

## Smaller things left open

Stages 3 and 4 of the positions work: ranged weapons in the combat loop and in the gear
chooser, after which `MOVEMENT_GATES_ATTACK` and `FIELD_LINGER` can retire. The
dex-skills half of the armour rule, deferred until `skill-list` carries governing
attributes as data. `SKILL_ATTRIBUTE` in `sim/model.py` hardcodes game data, which is a
single-source violation that predates this work and is still there. The tight-space
penalty stays unmeasurable until the model has walls.

## Reading order for a session starting cold

`CLAUDE.md` for the shape of the three repositories and the standing conventions,
`rules/ico/TODO.md` for every open question, and `rules/ico/sim/README.md` for what the
gates measure and why the baseline has moved. Those three are enough to pick the work
up; the transcript adds detail but nothing load-bearing.
