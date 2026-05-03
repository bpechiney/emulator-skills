Skills are organized into bucket folders under `skills/`:

- `engineering/` — daily code work
- `productivity/` — daily non-code workflow tools
- `misc/` — kept around but rarely used
- `personal/` — tied to my own setup, not promoted
- `deprecated/` — no longer used

The plugin manifest (`.claude-plugin/plugin.json`) lists the *active* set of skills. The bucket folders (`engineering/`, `productivity/`, `misc/`) hold the *available* set — they may contain inactive skills preserved from upstream that are not currently in the manifest. Inactive skills are documented in the top-level `README.md`'s "Inactive (preserved from upstream)" section. Skills in `personal/` and `deprecated/` must not appear in the manifest, the top-level `README.md`, or the bucket `README.md`s.

Each active skill in `engineering/`, `productivity/`, or `misc/` must have a reference in the top-level `README.md`. Each skill entry in the top-level `README.md` must link the skill name to its `SKILL.md`.

Each bucket folder has a `README.md` that lists every skill in the bucket (active or inactive) with a one-line description, with the skill name linked to its `SKILL.md`. Inactive skills are flagged with `(inactive)`.

This is a fork of [mattpocock/skills](https://github.com/mattpocock/skills) specialized for cycle-accurate retro console emulator development in Zig. The fork's signature skill is `engineering/emudev`.
