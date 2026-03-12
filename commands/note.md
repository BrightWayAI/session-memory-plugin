---
description: Quick one-liner capture to a project node. Lighter than /remember (no extraction process) and lighter than /learn (no knowledge typing). Just append a timestamped note to the changelog. Use for quick facts, status updates, or things worth noting that don't need structure.
---

# /note [node] [content]

The fastest way to commit something to memory. One line, no ceremony.

---

## Usage patterns

```
/note client:acme Kim confirmed the March 15 deadline
/note bizdev Had intro call with Stripe partnership team — they're interested
/note hiring Sent offer letter to Jordan for the ops manager role
/note strategy:pricing Competitor just dropped their entry tier to $29/mo
/note company-ops Renewed Slack and Notion — annual contracts locked through 2027
/note learning:ai-tools Finished testing Cursor vs Claude Code — notes in the shared doc
```

---

## Step 1 — Parse

Extract:
- **Node**: Which project node (required)
- **Content**: The note (everything after the node)

If node is ambiguous or missing, infer from conversation context or ask.

---

## Step 2 — Write

Append a lightweight LOG entry:

```
[node-id] LOG YYYY-MM-DD — Note: [content]
```

Do NOT update the living summary unless the note represents a significant state change (e.g. a project completing, a major blocker resolving).

---

## Step 3 — Confirm

One-line confirmation:
```
Noted in [node-id]: [content]
```

That's it. No analysis, no follow-up questions.

---

## Behavior notes

- Absolute minimum friction. This should feel like jotting something on a sticky note.
- No extraction, no knowledge typing, no continuity check.
- If the user streams multiple `/note` calls, handle each independently.
- If the content looks like it should be a knowledge entry (gotcha, lesson, model), suggest: "This sounds like a [gotcha/lesson/model] — want me to `/learn` it instead so it's easier to find later?"
