---
name: end-day
description: End-of-day closing routine. Recaps today, prompts for reflection (what got done, what blocked you, tomorrow's priority), commits learnings to memory, optionally pre-stages tomorrow via plan-tomorrow. Auto-fires on "/end-day", "wrapping up", "calling it a day", "done for today", "end of day", "closing out today", or any phrase signaling the workday is ending. Takes ~5 minutes.
---

See `commands/end-day.md` for the full workflow.

## When this skill fires

- User runs `/end-day` directly
- User says: "wrapping up", "calling it a day", "done for today", "end of day", "closing out", "before I sign off"
- A scheduled task triggers this (when wired up in Cowork — common pattern: weekday 4:30pm)

## Pre-flight

This skill works best with `claude-cortex` for memory commits and (optionally) `plan-tomorrow` for staging the next day. It works without them, but the value of the daily reflection compounds when memory captures it.

## What this skill is NOT for

- Mid-day status. Use `/recall` or `/search` instead.
- Long retrospectives. That's `/end-week`.
- Verbose summaries. The closing ritual is light, not exhaustive.
