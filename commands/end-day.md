---
description: Close out the work day. Reviews what happened today (calendar, memory commits, pipeline activity), prompts for end-of-day reflection (what got done, what blocked you, what's tomorrow's priority), commits learnings to memory, and optionally pre-stages tomorrow's plan via plan-tomorrow.
---

# /end-day

End-of-day closing routine. Closes the loop on today: capture what happened, what was learned, what blocked you, then commit it all to memory and (optionally) pre-stage tomorrow.

This is one of two "closing loop" skills (the other is `/end-week`). Run it once per work day, ideally late afternoon (4–5pm). Takes ~5 minutes.

---

## Step 1 — Recap today

Read today's calendar (working hours window). Surface:

- **External meetings held** — count, attendees, ~~one-line outcome each (ask user to fill)~~
- **Task blocks executed** — from any blocks scheduled by `/plan-tomorrow` or other planning, did they happen as planned?
- **Unscheduled time** — any large unstructured blocks the user might have used productively or lost

If the user has `claude-cortex` and active project nodes, also surface:

- **Active projects touched** — which nodes were referenced or worked on today (you can infer from the conversation, calendar, or ask)

Format the recap as a short, scannable summary. Don't dump the whole day's calendar verbatim — synthesize.

---

## Step 2 — Reflective prompts

Ask the user, one at a time:

1. **"Biggest thing that got done today?"** — capture as a LESSON or INSIGHT depending on shape. Goes to memory under the relevant project node.
2. **"What blocked you, if anything?"** — capture as a GOTCHA if it was a structural block, or as a BLOCKER in the relevant project's open threads.
3. **"Anything you learned about a person, project, or process?"** — capture as a knowledge entry (PEOPLE / MODEL / GOTCHA per type).
4. **"What's the one thing tomorrow has to move?"** — capture as a `[P0]` next-action in the relevant project node, with date stamp.

Keep these conversational. Don't make the user fill out a form. If they say "nothing major today," that's a valid answer — skip to Step 3 without padding.

---

## Step 3 — Commit to memory

Run the equivalent of `/remember` silently — extract the answers from Step 2 and commit them:

- LESSON / INSIGHT / MODEL / GOTCHA / RECIPE / PEOPLE entries → relevant project node files
- P0/P1 next-actions → project node `## Next Actions`
- Open thread updates → project node `## Open Threads`
- Pattern/preference observations → `~/Documents/Claude/memory/user.md`

If multiple project nodes are touched, commit to each. Update `DASHBOARD.md` with the latest activity.

Confirm briefly: "Committed [N] entries across [M] nodes."

---

## Step 4 — Pre-stage tomorrow (optional)

If the user has `plan-tomorrow` installed, offer:

> "Want me to run `/plan-tomorrow` now while context is fresh? It'll pull tomorrow's CRM tasks, memory P0s, and inbox action items, then propose calendar blocks. You can confirm before anything's written."

If yes → invoke `/plan-tomorrow`'s workflow.
If no → skip; user can run it tomorrow morning.

---

## Step 5 — Close

Confirm completion briefly:

> "Day closed. [N] learnings captured. [If plan-tomorrow ran:] [M] blocks staged for [tomorrow's date]. See you tomorrow."

Don't summarize verbosely. Closing should feel light.

---

## Behavior rules

- **Conversational, not formal.** This is a reflection ritual, not a status report.
- **Skip what doesn't apply.** If the user says nothing major happened, skip the reflection and just confirm the day is closed.
- **Don't over-capture.** Trivial things don't go in memory. The bar is "would I want to remember this in 3 months?"
- **Honor the user's pace.** If they want to skip Step 4 (plan-tomorrow), respect it. The closing ritual works without daily planning attached.

## When this skill is NOT the right fit

- **Mid-day check-ins.** This is end-of-day. For mid-day status, use `/recall [project]` or `/search`.
- **Long retrospectives.** Use `/end-week` (or cortex's `/review`) for synthesized weekly reflection.
- **Session memory dumps.** That's `/remember`. `/end-day` is structured around the *day's rhythm*, not the conversation.
