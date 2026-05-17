---
name: start-nucleus
description: >
  Foundational onboarding walker for Nucleus. Chains the essential setup
  commands needed for the marketplace to be productive — identity, voice,
  note sources, Obsidian vault, per-plugin setups, diagnostics, optional
  schedule registration. Idempotent.

  Auto-fires when the user says: "start nucleus", "let's get started",
  "set me up", "onboard me", "first time setup", "let's begin", "I just
  installed Nucleus, now what", "configure everything", "walk me through
  setup". The natural phrase a new user types when they don't yet know
  what commands exist.

  Goes from zero to working Nucleus in ~15-30 minutes depending on how
  many plugins are installed and which steps the user skips. Safe to
  re-run any time — completed steps are silently skipped.
---

# start-nucleus

See `commands/start-nucleus.md` for the full workflow.

## When to fire

- Explicit: `/start-nucleus`, or any trigger phrase above.
- Implicit: if a session starts and `<config-root>/identity.md` doesn't exist AND the user's first utterance is open-ended ("hello", "let's begin"), the cortex auto-recall layer should suggest: "Looks like Nucleus isn't set up yet. Want me to walk you through it? Run /start-nucleus."

## Pre-flight

Nothing required. This is the entry point — it bootstraps everything else. The only assumption is that the cortex plugin (which owns this command) is installed.

## What this skill is NOT for

- Re-setup of already-configured installations (just re-run individual `/setup-X` commands).
- New-plugin onboarding when Nucleus is already set up (run that plugin's `/setup-X` directly).
- Switching between configurations (no support for multi-profile setups in v0.1).
- Updating identity / voice / etc. after they're captured (re-run the individual setup commands).
