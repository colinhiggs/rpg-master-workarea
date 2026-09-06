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