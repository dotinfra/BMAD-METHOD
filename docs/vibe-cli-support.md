---
title: "Vibe CLI Support"
description: Plan and replay layer for Mistral Vibe support in this fork: install, subagents, documentation, and reapplication after every upstream sync.
---

# Vibe CLI Support

This page plans Mistral Vibe (Vibe CLI and Vibe Code) support in this fork: what already works, the deltas this branch carries, the project files the layer generates, and how to replay everything after an upstream sync.

## What Already Works

The V7 installer is the Skills CLI, and it already ships a Mistral Vibe agent: id `mistral-vibe`, project skills in `.vibe/skills/`, global skills in `~/.vibe/skills/`, and `VIBE_HOME` respected. Install with `-a mistral-vibe` or pick it from the prompt. On V7 the installation path needs documentation only; no installer change is required.

## How Vibe Subagents Work

Vibe defines subagents as TOML files, not in-session personas.

| Aspect | Behavior |
| --- | --- |
| Definition | TOML file in `.vibe/agents/` (project) or `~/.vibe/agents/` (user) |
| Type | `agent_type = "subagent"` |
| Prompt | `system_prompt_id` points at a file under `.vibe/prompts/` |
| Selection | The `description` field drives delegation |
| Tools | `enabled_tools` lists built-in tools only, no connectors or MCP |
| Skills | `enabled_skills` filters the skills an agent can load, globs allowed |
| Result | Text only, no questions to the user mid-task |
| Concurrency | Up to 32 at once, gated by `enable_subagents` in `config.toml` |

Verify the exact field names against the installed Vibe version before generating files; the generator writes the minimal set above so a rename is a one-line fix.

## How BMad Maps to Vibe

| BMad mechanism | V7 instruction today | Vibe outcome |
| --- | --- | --- |
| Party `subagent` mode | `mode-subagent.md` spawns one agent per persona and defines a fresh-per-round fallback where the harness cannot hold agents between turns | Works as-is: fresh spawn each round, room context carried in the brief |
| Party `agent-team` mode | `mode-agent-team.md` is Claude Code only, with fallback to `subagent` then `session` | Vibe uses the documented fallback: `subagent` mode |
| `bmad-build-auto` subagents | `workflow.md` launches every subagent of a step as blocking calls in one message, never detached | Matches Vibe delegation: launch all, await together, 32 limit far above lens counts |
| Review lenses (step 4) | A lens receives content or a path in its launch message and returns findings as JSON text | Matches the Vibe text-only contract; lens files stay unchanged |
| Full-route patch loop | `step-04-review.md` re-engages the step 3 subagent and falls back to applying the patches yourself when it cannot be continued | Vibe cannot continue a subagent, so the documented fallback applies |

Two points to verify in a live session: whether a Vibe subagent TOML pins a model, and which built-in tools (web search in particular) a subagent can use.

## What This Branch Changes

| Change | Place | Upstream PR |
| --- | --- | --- |
| Name the `mistral-vibe` agent in the V7 install section | `README.md` | Yes |
| Add a Mistral Vibe row to the skills directory table | `docs/reference/skills-and-agents.md` | Yes |
| Note that Vibe has no team mechanism and falls to `subagent` | `mode-agent-team.md` | Yes |
| Replay script for all deltas and project generation | `tools/vibe/apply_vibe_support.py` | Fork layer |
| Vibe skill that drives the replay | `tools/vibe/skill/bmad-vibe-sync/SKILL.md` | Fork layer |
| Generated subagent files | `.vibe/agents/`, `.vibe/prompts/` in each project | Not in the repo |

## Generated Project Files

`--apply-project` writes into a project that has BMad installed under `.vibe/skills/`:

| File | Source |
| --- | --- |
| `.vibe/agents/bmad-party-<persona>.toml` and its prompt | Each installed `bmad-agent-*` skill; the prompt is its `SKILL.md` |
| `.vibe/agents/bmad-review-<lens>.toml` and its prompt | `bmad-build-auto/review-prompts/*.md` |
| `.vibe/agents/bmad-implementer.toml` and its prompt | Fixed implementer prompt for `bmad-build-auto` step 3 |

Review agents get `enabled_skills = ["bmad-build-auto"]`; persona prompts carry the whole persona so the orchestrator only adds the room context.

## V6.12 Overlay

V6.12 is frozen, so no upstream PR is expected and support stays a fork overlay. Install with an existing channel, then run `--overlay-v612` with `--channel agents`: it copies the skill folders into `.vibe/skills/` and generates the same agent files. Re-run after every `npx bmad-method install`; it is idempotent.

## Replay After an Upstream Sync

1. Sync upstream into `main` and `dev` as usual, then rebase this branch.
2. Run `uv run tools/vibe/apply_vibe_support.py --check` from the fork root. Each delta reports APPLIED, MISSING, or DRIFT.
3. `--apply-repo` re-applies MISSING deltas. DRIFT means upstream text moved: apply that delta by hand, then update its anchor in the script so the next sync finds it.
4. For each project, re-run `--apply-project` (V7) or `--overlay-v612` (V6.12).
5. Validate: `uv run tools/validate_skills.py --strict`, `uv run --frozen python -B -m pytest`, and in `docs-site` run `npm run fix-links`, `npm run validate-links`, and `npm run build`. If the docs build requires the page in the sidebar, add it under Reference.

## Upstream Path

The PR targets `dev`, never `main`. Conventional Commits, no tool or model attribution anywhere. The three documentation deltas are small and standalone; send them on their own if the maintainers prefer to keep the fork layer out.

## Session Checklist

1. Verify the subagent TOML fields against the installed Vibe version; adjust the generator if a field name differs.
2. Run `uv run tools/vibe/apply_vibe_support.py --apply-repo` and confirm `--check` reports all APPLIED.
3. In a scratch project: `npx skills add dotinfra/BMAD-METHOD/skills/core-tools --skill '*' -a mistral-vibe` and the same for `skills/method`, run `bmad setup`, then `--apply-project`.
4. Run one party in `subagent` mode and one `bmad-build-auto` oneshot with `review quick`; confirm delegation and JSON lens output.
5. Validate the repo as in Replay step 5, then commit on this branch.
