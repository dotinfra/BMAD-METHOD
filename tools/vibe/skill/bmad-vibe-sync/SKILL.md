---
name: bmad-vibe-sync
description: "Re-applies Vibe CLI support after an upstream sync: checks the replay anchors, re-applies drifted repo deltas, and regenerates the Vibe subagent files of a BMad project. Use when asked to sync, reapply, or repair Vibe support in this fork or project"
---

# Vibe Sync

Re-apply the Vibe CLI support layer of this fork. Three targets: the repo deltas, a project install, and a V6.12 overlay.

## Repo Deltas

From the fork root, run `uv run tools/vibe/apply_vibe_support.py --check`. Each line is APPLIED, MISSING, or DRIFT.

- APPLIED: nothing to do.
- MISSING: run `uv run tools/vibe/apply_vibe_support.py --apply-repo` to apply every missing delta.
- DRIFT: upstream text moved. Open the file, find the anchor described in `docs/vibe-cli-support.md`, apply the addition by hand at its new place, then update the anchor in the script so the next sync finds it. Never guess a location.

## Project Generation

For a project with BMad installed under `.vibe/skills/`, run `uv run tools/vibe/apply_vibe_support.py --apply-project <project-root>`. It writes `.vibe/agents/*.toml` and `.vibe/prompts/*.md` for party personas, review lenses, and the implementer. Re-run after every install or update; it overwrites only what it generates.

## V6.12 Overlay

For a V6.12 project, install with an existing channel, then run `uv run tools/vibe/apply_vibe_support.py --overlay-v612 <project-root> --channel agents`. It copies the skill folders into `.vibe/skills/` and generates the same agent files.

## Validate

After repo changes, run `uv run tools/validate_skills.py --strict` and `uv run --frozen python -B -m pytest`. In `docs-site`, run `npm run fix-links`, `npm run validate-links`, and `npm run build`.
