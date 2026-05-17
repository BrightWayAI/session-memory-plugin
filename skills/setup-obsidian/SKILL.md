---
name: setup-obsidian
description: >
  Scaffold an Obsidian vault config over `<config-root>/`. After this runs, the
  user can open Obsidian → File → Open vault → select <config-root>/ and see
  their cortex memory as a graph, with daily-brief snapshots as daily notes and
  a VAULT.md home page powered by Dataview.

  Auto-fires when the user says: "set up Obsidian", "make this work in
  Obsidian", "enable graph view", "make this work on mobile", "I want to use
  Obsidian", "set up a vault", "configure Obsidian", "open this in Obsidian".

  Idempotent and non-destructive — existing `.obsidian/` files and VAULT.md
  are preserved on re-run. Force-reset available via `/setup-obsidian --reset`.
---

# setup-obsidian

See `commands/setup-obsidian.md` for the full workflow.

## When to fire

- Explicit: `/setup-obsidian` or any trigger phrase above.
- Implicit cross-skill: if `/setup-identity` finishes and the user mentions Obsidian or mobile access in the conversation, offer to chain into `/setup-obsidian`.

## What to do

1. Resolve `<config-root>` and read identity for personalization.
2. Confirm the plan (which files will be written, which preserved).
3. Write `.obsidian/app.json`, `core-plugins.json`, `daily-notes.json` (skip existing).
4. Write `VAULT.md` (skip existing).
5. Inform about daily-brief and memory-index integration status.
6. Report.

## What this skill does NOT do

- Does not install Obsidian.
- Does not install Obsidian community plugins (it lists them, user clicks through).
- Does not modify any other plugin's behavior. Pure config scaffolding.
- Does not overwrite existing user files unless `--reset` is explicitly passed and re-confirmed per file.
