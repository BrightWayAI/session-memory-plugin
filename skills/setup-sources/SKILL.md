---
name: setup-sources
description: Configure cortex's note-source list — Granola, Gemini, Fireflies, Otter, Notion, generic Drive folders or Gmail labels, custom. Auto-fires on "/setup-sources", "set up cortex sources", "configure my notetakers", "set up the mining layer", "add a notetaker to cortex", "configure note sources for end-day", or when /end-day reports "no note sources configured."
---

See `commands/setup-sources.md` for the full interview.

## When this skill fires

- User runs `/setup-sources` directly
- User says: "set up cortex sources", "configure my notetakers", "set up the mining layer", "add a notetaker"
- `/end-day` reports "no note sources configured" and offers to route into this command
- User adopts a new notetaker (e.g., adds Fireflies to a specific client) and wants to register it

## What this skill is NOT for

- **Connector authentication** — connecting Granola / Gmail / Drive / Notion MCPs is owned by the Cowork / Claude Code runtime. This skill assumes connectors are already wired and only configures cortex's source list against them.
- **Notetaker software setup** — installing the Granola app, configuring Gemini in Google Meet, etc. is out of scope. Configure the tool first, then run this command to register it with cortex.
- **Replacing the adapter reference** — the per-provider fetch logic lives at `agents/lib/note-source-adapters.md`. This command writes the user's source-list config; the adapter file decides how each provider is queried.

## Inputs

- `<config-root>/plugins/cortex.note-sources.md` (read-modify-write; created if missing)
- `agents/lib/note-source-adapters.md` (read-only — the per-provider interview questions and health-checks come from here)

## Outputs

- `<config-root>/plugins/cortex.note-sources.md` written or updated
- Health-check results surfaced inline during setup so misconfigurations fail loud here, not silent at `/end-day` time

## Behavior rules

- Idempotent
- Loud failure on adapter health-check failures
- Never write a broken config (drop failed-health sources rather than save with errors)
