# Emudev Skills

Agent skills for building cycle-accurate retro console emulators (Game Boy, NES, SNES) in Zig 0.16.

A specialized fork of [mattpocock/skills](https://github.com/mattpocock/skills). The signature skill is **[`/emudev`](./skills/engineering/emudev/SKILL.md)** — coding standards and citation discipline for cycle-accurate emulator work. The remaining skills (TDD, grilling, diagnosis, issue management) compose with `/emudev` and are kept largely as-is from upstream.

## Quickstart

1. Install:

```bash
npx skills@latest add bpechiney/emulator-skills
```

2. In your emulator repo, run `/setup-matt-pocock-skills` once. This scaffolds `docs/agents/{issue-tracker,triage-labels,domain}.md` and adds an `## Agent skills` block to `AGENTS.md`/`CLAUDE.md`.

3. Then run `/emudev` (or trigger it by editing emulator code). On first invocation in a repo it lazily creates `docs/agents/emudev.md` via a short interview (system, cycle-accuracy tier, test-ROM root, `Hacks` location, Zig version) and amends the `## Agent skills` block.

## Why emulator dev needs its own skills

Cycle-accurate emulator code has failure modes generic engineering skills don't cover:

- **Citation discipline** — every hardware-derived code path needs a citation, or it's unreviewable. Generic Zig culture treats comments as a code smell; emulator code inverts this.
- **Hardware-quirk fidelity** — the HALT bug, sprite-0 timing, mode-3 length variance, OAM corruption-on-$2003-write, mid-frame palette writes, DMC sample-stealing CPU stalls. Each is a `QUIRK` tag with a citation; together they're the difference between a toy and a Tetris-runs-correctly emulator.
- **Test-ROM compliance** — Blargg, Mooneye, mealybug-tearoom, nestest, Tom Harte JSON tests. The "did I implement opcode 0x76 correctly" question has a deterministic answer, and the emulator's build pipeline must integrate that signal.
- **Hack-debt management** — bsnes/ares typed `Hacks` namespace from day one, not Snes9x-style untyped accumulation. A `HACK` tag without a game name, linked issue, and stated hardware uncertainty isn't a hack — it's bad code.
- **Save-state schema versioning** — bumping a version without a migration breaks every user's save file. Decide migration discipline before shipping, not after.
- **Fidelity scope** — DMG-only or DMG+CGB or all six revisions? NTSC-only or NTSC+PAL? 1-CHIP or 2/1/3-CHIP SNES PPU? Choose deliberately; gate revision-dependent paths.

The `/emudev` skill encodes these as standing rules and surfaces six load-bearing decisions (dispatch strategy, mapper polymorphism, cycle-accuracy tier, save-state versioning, CPU↔Bus boundary, fidelity scope) for `/grill-with-docs` to walk before implementation.

## Why these skills exist (carried over from upstream)

The fundamentals below apply to any non-trivial codebase, including emulators. Most of this section is preserved from mattpocock/skills with minor framing updates.

### #1: The agent didn't do what I want

> "No-one knows exactly what they want"
>
> David Thomas & Andrew Hunt, [The Pragmatic Programmer](https://www.amazon.co.uk/Pragmatic-Programmer-Anniversary-Journey-Mastery/dp/B0833F1T3V)

The most common failure mode is misalignment. The fix is a **grilling session**:

- [`/grill-me`](./skills/productivity/grill-me/SKILL.md) — for non-code uses
- [`/grill-with-docs`](./skills/engineering/grill-with-docs/SKILL.md) — same, but updates `CONTEXT.md` and ADRs inline

Use them every time you start a new feature. For emulator work, run `/grill-with-docs` against each of the six load-bearing decisions named in `/emudev`.

### #2: The agent is way too verbose

> With a ubiquitous language, conversations among developers and expressions of the code are all derived from the same domain model.
>
> Eric Evans, [Domain-Driven Design](https://www.amazon.co.uk/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)

A shared `CONTEXT.md` decodes domain jargon. Built into [`/grill-with-docs`](./skills/engineering/grill-with-docs/SKILL.md). Emulator domains are jargon-heavy (cycles, scanlines, mappers, T-states, hblank, vblank, OAM) — `CONTEXT.md` is essential, not optional.

### #3: The code doesn't work

> "Always take small, deliberate steps. The rate of feedback is your speed limit."
>
> David Thomas & Andrew Hunt, *The Pragmatic Programmer*

Use:

- [`/tdd`](./skills/engineering/tdd/SKILL.md) — red-green-refactor with vertical slices. For emulator work, it consumes test ROMs as integration fixtures (see `/emudev`'s `testing.md`).
- [`/diagnose`](./skills/engineering/diagnose/SKILL.md) — disciplined bug-hunt loop. Emulator dev is bug-hunt-heavy; `/diagnose` is your daily driver when "Pokémon Red hangs at the intro" turns up.

### #4: We built a ball of mud

> "The best modules are deep. They allow a lot of functionality to be accessed through a simple interface."
>
> John Ousterhout, [A Philosophy of Software Design](https://www.amazon.co.uk/Philosophy-Software-Design-2nd/dp/173210221X)

Use:

- [`/to-prd`](./skills/engineering/to-prd/SKILL.md) — turns conversation context into a PRD
- [`/zoom-out`](./skills/engineering/zoom-out/SKILL.md) — gives a module map for unfamiliar code
- [`/improve-codebase-architecture`](./skills/engineering/improve-codebase-architecture/SKILL.md) — finds deepening opportunities, informed by `CONTEXT.md` and ADRs

## Reference (active manifest)

### Engineering

- **[emudev](./skills/engineering/emudev/SKILL.md)** — Coding standards for cycle-accurate retro console emulators in Zig 0.16. Standing rules (six-tag citation taxonomy, no-alloc hot path, typed `Hacks`, `TODO`/`HACK` linked-issue), six load-bearing decisions for `/grill-with-docs`, code patterns (labeled-switch dispatch, tagged-union polymorphism, packed-struct register layouts), test discipline (determinism, save-state round-trip).
- **[diagnose](./skills/engineering/diagnose/SKILL.md)** — Disciplined diagnosis loop: reproduce → minimise → hypothesise → instrument → fix → regression-test.
- **[emulator-code-review](./skills/engineering/emulator-code-review/SKILL.md)** — Reviews Zig code for cycle-accurate Game Boy, NES, and SNES emulators. Per-component checklists (CPU, PPU, APU, bus, mapper, save-state, cartridge loader, test-ROM harness), cycle-accuracy bug catalogue, citation rubric (✅⚠️❌), confidence-with-test-ROM-evidence output. Pairs with `/emudev` as the review-side counterpart — `/emudev` writes the code; this skill reviews it.
- **[emulator-diagnosis](./skills/engineering/emulator-diagnosis/SKILL.md)** — Symptom-driven diagnosis for cycle-accurate Game Boy, NES, and SNES emulators. Maps observed symptoms (boot freezes, audio glitches, save-state corruption, region desync) to likely causes by category. Use when starting from a bug report rather than a code area.
- **[grill-with-docs](./skills/engineering/grill-with-docs/SKILL.md)** — Grilling session that challenges your plan against the existing domain model and updates `CONTEXT.md` / ADRs inline.
- **[improve-codebase-architecture](./skills/engineering/improve-codebase-architecture/SKILL.md)** — Find deepening opportunities, informed by `CONTEXT.md` and `docs/adr/`.
- **[setup-matt-pocock-skills](./skills/engineering/setup-matt-pocock-skills/SKILL.md)** — Scaffold the per-repo config (issue tracker, triage labels, domain doc layout). Run once per repo before the other skills.
- **[tdd](./skills/engineering/tdd/SKILL.md)** — Test-driven development with red-green-refactor. Drives the loop emudev defers to.
- **[to-issues](./skills/engineering/to-issues/SKILL.md)** — Break a plan / spec / PRD into independently-grabbable issues using vertical slices.
- **[to-prd](./skills/engineering/to-prd/SKILL.md)** — Turn the current conversation context into a PRD on the issue tracker.
- **[triage](./skills/engineering/triage/SKILL.md)** — Triage issues through a state machine of triage roles.
- **[zoom-out](./skills/engineering/zoom-out/SKILL.md)** — Get a higher-level map of an unfamiliar section of code.

### Productivity

- **[grill-me](./skills/productivity/grill-me/SKILL.md)** — Get relentlessly interviewed about a plan or design until every branch is resolved.
- **[write-a-skill](./skills/productivity/write-a-skill/SKILL.md)** — Author new skills with proper structure and progressive disclosure.

### Misc

- **[git-guardrails-claude-code](./skills/misc/git-guardrails-claude-code/SKILL.md)** — PreToolUse hook blocking dangerous git commands (`push --force`, `reset --hard`, `clean -f`, `branch -D`).

## Inactive (preserved from upstream)

These skills are present in the source tree but excluded from `.claude-plugin/plugin.json`. They are kept for low-friction upstream rebase, not for active use in this fork. Re-add to `plugin.json` if you find a use for one.

- `productivity/caveman` — ultra-compressed communication mode. Not used here; full context budgets are fine.
- `misc/setup-pre-commit` — Husky / lint-staged / Prettier setup. TypeScript-shaped; this fork uses Zig with `nix develop -c zig fmt`.
- `misc/migrate-to-shoehorn` — TypeScript-specific migration helper.
- `misc/scaffold-exercises` — AI-Hero-CLI course-tooling scaffold.

`personal/` and `deprecated/` skills are excluded from this list and from the manifest by repo convention.

## Credit

Forked from [mattpocock/skills](https://github.com/mattpocock/skills). The "Why these skills exist" framing, the grilling/TDD/triage/PRD/setup skills, and the bucket structure are Matt's. Upstream changes are pulled when relevant; the fork specializes for cycle-accurate emulator development.
