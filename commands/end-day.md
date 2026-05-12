---
description: End-of-day orchestration ritual (v4.2+). Chains five steps with user gates between them — inbox triage for tomorrow, transcript review for uncaptured commitments, cortex auto-commit with Phase 4 cheap-tier triage, reflective prompts, and pre-stage tomorrow's brief artifact. Run once per work day, late afternoon. Takes 10-15 minutes including user-gated decisions.
---

# /end-day

End-of-day closing chain. Reads the day, captures commitments, updates memory with cost discipline, and pre-stages tomorrow so the morning has a working surface waiting.

This rewrite (v4.2) ties together the second-brain v2 phases — inbox triage, transcript review, person-page graduation via cheap-tier commit triage, daily-brief pre-stage. Each step has a user gate so the chain never blocks on something the user wants to defer.

Run once per work day, ideally 4-6pm. If a downstream plugin isn't installed (e.g., `daily-brief` not yet adopted), the chain skips that step silently and continues.

---

## Pre-chain — Resolve config root and identity

Resolve `<config-root>` via `~/Documents/.claude-plugin-config-root` (platform-aware Step 0 — see any setup command for the pattern). Read `<config-root>/identity.md` for time zone (defines "today" / "tomorrow").

Determine `today_local` and `tomorrow_local` (next business day: Mon-Thu → tomorrow; Fri → Monday; Sat/Sun → Monday).

---

## Pre-chain B — One-time Scope migration (v4.3+)

The mining layer (Steps 2a / 2b below) routes extracted content to nodes using a Scope convention on each domain-shaped node. This migration runs **once** to adopt the convention on existing nodes. After it completes, a marker file at `<config-root>/memory/.scope-migration-done` prevents re-prompting.

Skip this block if `<config-root>/memory/.scope-migration-done` exists. Otherwise:

### A — Detect domain-shaped nodes

Scan `<config-root>/memory/`. A node is "domain-shaped" if ALL of:

- It's a `.md` file at root level (e.g., `<config-root>/memory/<name>.md`), OR a `.md` file in a non-reserved subdirectory
- Its filename is NOT one of: `user.md`, `DASHBOARD.md`, `triage-log.md`, `CLAUDE.md`
- Its containing directory is NOT one of: `client/`, `bizdev/`, `person/`, `archive/` (these are project / prospect / person / archive — not domain)
- Its YAML front-matter (if present) does NOT set `type: project` (engagement state, handled separately)

The detection is purely structural. The plugin does not know or care what specific domains the user has — it works against whatever nodes exist.

### B — Synthesize a Scope draft per detected node

For each detected node, read its current content and produce a draft Scope section pre-filled from what's already in the file:

- **Topics:** lowercased keywords inferred from existing knowledge-entry tags, recurring nouns in the Summary, and recurring section headers
- **Aliases:** alternate names mentioned in PEOPLE entries or Summary (e.g., a node named `studio.md` whose Summary references "Learning Production System" gets that as an alias)
- **What goes here:** one-line synthesis from the Summary section
- **What does NOT go here:** left empty in the draft — this is the most useful field but requires user judgment

### C — Per-node confirmation

Present each draft to the user, one node at a time:

> "Node `<name>`. Proposed Scope section:
>
> [draft]
>
> (a)ccept / (e)dit / (s)kip"

- `accept` → insert the Scope section as the first section of the node file (above existing Summary). Idempotent: if a Scope section already exists, replace it; if not, insert.
- `edit` → drop into edit mode. User edits inline, then accept.
- `skip` → skip this node. It can still receive routed content from miners, but routing accuracy drops without the Scope hint. The migration will not re-prompt for this node — user can re-run `/end-day` with `--rerun-scope-migration` later to revisit.

### D — Offer new domain nodes (open-ended)

After all detected nodes are processed, ask once:

> "Want to create any new domain nodes now? (open-ended — list them comma-separated, or skip)"

For each name the user lists:
1. Confirm node ID (kebab-case, lowercased)
2. Walk through the same Scope interview (topics / aliases / what goes here / what does NOT go here — all four asked, no draft to pre-fill since the node is new)
3. Create the node file at `<config-root>/memory/<name>.md` with the Scope section, an empty Summary placeholder, and standard sections (Knowledge / People / Changelog / Open Threads / Next Actions)

The plugin does not suggest names. The user drives the list.

### E — Mark complete

Write `<config-root>/memory/.scope-migration-done` with the ISO timestamp and the list of nodes processed (one line per node, `<status> <node-id>` — accepted, edited, skipped, or created). This file is the only check that prevents re-prompting on subsequent `/end-day` runs.

If the user runs `/end-day --rerun-scope-migration`, delete the marker and re-run this block.

### F — On failure

If Pre-chain B errors partway through (e.g., a node file is unreadable), log the partial state to `<config-root>/memory/.scope-migration-partial` and prompt the user: "Scope migration hit an error on `<node>`. Resume later via `/end-day --rerun-scope-migration`? Or continue with the rest of `/end-day` now using whatever Scope sections were captured?" Default: continue.

---

## Step 1 — Inbox triage for tomorrow

**Goal:** surface threads that will need a reply tomorrow so they can populate tomorrow's brief.

Behavior:

- If the user has the `inbox-triage` plugin installed and its `/triage-inbox` command is available → invoke it. The skill returns the top 3-7 Needs-reply-today threads (the classifier doesn't distinguish "today" from "tomorrow" — it returns "needs a reply in the next 24h," which from a 5pm vantage means tomorrow morning).
- If `inbox-triage` is NOT installed → fall back to a lighter Gmail search: messages received today where the user is in To/Cc and hasn't replied. Cap at 7.
- Either way, **stash the result** in memory for Step 5's pre-stage call. Don't render it to the user yet.

### User gate after Step 1

> "Tomorrow's inbox triage shows [N] threads. Want to review them now, or wait until morning when the brief surfaces them?"

- "Review now" → render the list and pause for user reactions before proceeding to Step 2
- "Wait" (default if no response within ~10s in interactive mode) → continue to Step 2

---

## Step 2 — Transcript review (v4.3+: two streams)

**Goal:** surface (a) commitments the user made today that aren't already in CRM tasks or cortex memory, and (b) learnings from the day's transcripts that aren't already captured in node content.

Read `<config-root>/plugins/cortex.note-sources.md` to get the configured note-sources list. Filter to enabled sources whose scope (global or `project:<node-id>`) matches the run's relevant nodes (sources scoped to a project node are included if that project has meetings in today's window).

If no sources are configured → skip this step with a one-line note: "No note sources configured. Run `/setup-sources` to enable transcript mining." Proceed to Step 2a.

Invoke `transcript-reviewer` with:
- `time_window: 1 day`
- `note_sources: [<filtered list>]`
- `node_inventory`, `node_summaries`, `dashboard_snapshot` (built from `<config-root>/memory/`)
- `crm_task_snapshot`: current HubSpot tasks owned by the user

The agent returns two streams: `commitments_delta` and `learnings_delta`.

### Commitments stream — user gate (unchanged from pre-v4.3)

For each item in `commitments_delta`, prompt per item:

> "Convert this to a CRM task / cortex P0 / both / skip?"

- **CRM task** → create via HubSpot MCP, due tomorrow (or a user-specified date), assigned to the user
- **Cortex P0** → append to the relevant project node's `## Next Actions` as `[P0]`
- **Both** → both side-effects
- **Skip** → no action, no log

If the delta is empty ("Nothing missing — all commitments already tracked"), confirm and continue. Don't pad.

### Learnings stream — DEFERRED to Step 2b unified gate

The `learnings_delta` stream is NOT reviewed here. Hold the proposals; they get merged with `conversation-miner` and `activity-miner` output in Step 2a, then reviewed once in Step 2b. This avoids review-fatigue and lets cross-source dedup happen across miners.

---

## Step 2a — Parallel mining of non-transcript sources (v4.3+)

**Goal:** mine the day's other Cowork sessions and CRM/email/calendar events for learnings that aren't yet in node content.

Run the following agents **in parallel** (one chat message, multiple Task tool blocks):

1. `conversation-miner` with:
   - `time_window: 1 day`
   - `current_session_id`: the session running `/end-day` (exclude from mining)
   - `node_inventory`, `node_summaries`, `dashboard_snapshot`
   - `user_email`, `user_local_tz` (from identity.md)

2. `activity-miner` with:
   - `time_window: 1 day`
   - `node_inventory`, `node_summaries`, `dashboard_snapshot`
   - `user_email`, `user_local_tz`

`code-miner` is deferred to a later cortex version (not built in v4.3).

Each miner runs its own cheap-tier triage gate first. Miners that triage-skip return empty and cost ~nothing.

### Merge step

Combine three sources of proposals:

- `transcript-reviewer`'s `learnings_delta` from Step 2
- `conversation-miner`'s `learnings_delta`
- `activity-miner`'s `learnings_delta`

Apply cross-miner dedup:

- For each proposal in the conversation-miner stream, check against transcript-reviewer's stream — if a proposal on the same target_node has > 70% content overlap, drop the conversation-miner version (transcript is the source of record).
- For each proposal in activity-miner's stream, do the same against transcript-reviewer.
- Within each remaining set, dedup against existing node content (n-gram > 70% overlap → drop).

Group surviving proposals by `target_node`. Sort within each group by `confidence DESC, update_type` (corrections first, then decisions/insights, then gotchas/models, then relationship-context).

---

## Step 2b — Unified review gate (v4.3+)

Present the merged proposals once. Format:

```
Today's mining surfaced [N] proposed updates across [K] nodes:

  ▸ <node-id> (<count>)
      [<update_type>, <confidence>] <content>
        — <source>, <ref>
      [<update_type>, <confidence>] <content>
        — <source>, <ref>
        cross-ref → <other-node-id>: <one-line content of the linked proposal>
      ...

  ▸ <other-node-id> (<count>)
      ...

Review:
  (a)ccept all
  (s)elect per node — show one node at a time, accept/edit/skip per item
  (e)dit each — walk every item individually
  (h)igh-confidence only — show only high-confidence items, skip the rest
  (k)skip all
```

### Auto-expand cross-refs

When rendering a proposal that has `cross_ref: [<id>]`, render the linked proposal's `content` inline as a "cross-ref →" line under it (see format above). The user reviews both in context without having to jump.

### High-confidence toggle behavior

If the user picks `(h)`, render only proposals with `confidence: high`. The rest are deferred (logged as "deferred — review tomorrow" in the dismissal log — they re-surface in tomorrow's mining if still applicable; they don't get auto-committed and they don't permanently disappear).

### New-node-creation confirmation

If any proposal has `node_type: new` (with a `new_node_suggestion`), surface a dedicated confirmation BEFORE accepting any content into it:

> "Create new node `<id>`? Type: <type>. Scope: <one-line>. (y / edit-scope / n)"

If `y` → create the node file with a Scope section (interview the user inline for the 4 Scope fields), then accept the proposal into it.
If `edit-scope` → walk the 4-field Scope interview before creating.
If `n` → drop all proposals targeting that new node (or, on a per-proposal basis, ask whether to re-route each one to an existing node).

This makes taxonomy growth explicit rather than silent.

### Dismissal log

Skipped proposals (whether from `(k)skip all`, per-item skip, or implicit when `(h)high-confidence` was chosen) are logged to `<config-root>/memory/dismissed-proposals.log` (append-only), keyed by a hash of `(source.note_id or session_id or event_ref) + content` so they don't re-surface tomorrow. Format:

```
<ISO timestamp>  <proposal_id>  <target_node>  <update_type>  <source_kind>:<ref-hash>  <one-line content>
```

Miners read this log at the top of every run and skip any proposal whose source-ref + content hash matches a dismissed entry within the last 7 days. (After 7 days, re-surfacing is permitted — the user may have changed their mind.)

### Accepted proposals → Step 3

Accepted proposals (and edits) are passed forward to Step 3 as additional input alongside the current session's content.

---

## Step 3 — Cortex auto-commit with cheap-tier triage

**Goal:** capture the day's learnings, decisions, and observations to memory — without burning Sonnet tokens on trivial conversations.

Run `/remember` in silent mode with **two inputs**:

1. The current session's content (normal /remember source)
2. The list of accepted proposals from Step 2b — these are pre-routed (each has a `target_node` and `section`) so /remember treats them as already-classified rather than re-classifying through Step 0's Haiku triage. The triage runs ONLY on the current session content; accepted proposals bypass it (the user just accepted them, that's the commit-worthy signal).

**Step 0 (cheap-tier triage) is mandatory for the current-session content** — it's the whole reason the session half is cost-disciplined.

- Classifier decides commit-worthiness and node list for the current session
- Synthesis runs only on affected nodes
- Trivial day with no accepted proposals → `commit: false` on the session AND empty accepted-proposals list → one line in `triage-log.md`, no Sonnet call
- Substantive day OR accepted proposals exist → normal flow, with the Phase 3 person-page graduation logic firing where relevant

User-observation CORRECTIONs always commit to the `user` node regardless of the classifier's decision (see `skills/observe/SKILL.md` for the override rule).

When writing the proposals to their target nodes, follow the v4.3+ knowledge-entry tag convention (see `/remember` Step 3 C.0): every new entry gets `[confirmed:<today>] [recalled:<today>]` tags. Entries the mining layer **updates** (e.g., a re-affirmed insight) get `[confirmed:<today>]` while leaving `[recalled:...]` to be updated by the next /recall that surfaces them.

Confirm briefly to the user (not verbose):

> "Memory updated: [N] nodes touched, [M] knowledge entries committed, [K] person-page updates, [P] mined proposals applied."

If the classifier returned `commit: false` AND no proposals were accepted, say: "Quiet day — nothing material to commit. Logged the audit line." and continue.

---

## Step 4 — Reflective prompts

**Goal:** capture the human-readable version of what mattered today, separate from the structured commits in Step 3.

Ask the user, conversationally, one at a time:

1. **"Biggest thing that got done today?"** — captured as a LESSON or INSIGHT depending on shape, written to the relevant project node.
2. **"What blocked you, if anything?"** — captured as a GOTCHA if structural, or as a BLOCKER on the project's open threads.
3. **"What's the one thing tomorrow has to move?"** — captured as a `[P0]` next-action on the relevant project node, dated tomorrow.

These three answers also feed Step 5's pre-stage as section-6 content of tomorrow's brief (yesterday's reflection, from tomorrow's perspective).

Keep conversational. If the user says "nothing major today," that's valid — skip to Step 5.

If the user provides answers, **append them to today's brief markdown snapshot** at `<config-root>/briefs/<today_local>.md` under section 7 (End-of-day prompts). This is the canonical write — it ensures the snapshot reflects what was actually captured, even if `daily-brief` isn't installed.

---

## Step 5 — Pre-stage tomorrow's brief

**Goal:** when tomorrow morning hits, the brief is already waiting.

If the `daily-brief` plugin is installed:

1. Invoke its `/brief` command with `target_date: tomorrow_local`.
2. Pass the inbox-triage results from Step 1 directly (so the brief doesn't re-query Gmail).
3. Pass today's reflection answers from Step 4 to populate tomorrow's section 6 (Yesterday's reflection).
4. The brief generator writes `<config-root>/briefs/<tomorrow_local>.md` and updates the Cowork artifact "Today's Brief" to tomorrow's data — or, if Cowork artifact tools aren't available (Claude Code), produces the markdown snapshot only with a clear notice.

### User gate after Step 5

> "Tomorrow's brief is staged. Want to review it now, or wait until morning?"

- "Review now" → render section summaries inline (don't dump full sections — that's what the artifact / snapshot is for)
- "Wait" (default) → confirm and close

If `daily-brief` is NOT installed, skip Step 5 entirely. The chain still produced value (triage results visible in chat, commitments converted, memory committed, reflections captured to today's snapshot).

---

## Step 6 — Close

Confirm completion briefly:

> "Day closed. [N] commitments converted, [M] memory entries, [K] person pages touched. Tomorrow's brief staged for [tomorrow_local]. See you tomorrow."

Adjust the count summary based on what actually ran (don't fabricate counts for skipped steps).

---

## Default behavior on user-gate timeouts

If the user is running this in a fire-and-forget mode (e.g., via a scheduled task, or they walked away), apply these defaults:

- **Step 1 gate** → "Wait" after ~10s (default)
- **Step 2 commitments gate** → "Skip" per item after ~10s (don't auto-create CRM tasks without confirmation — destructive on the wrong side)
- **Step 2b unified review gate** → "Skip all" after ~30s. Never auto-commit mined proposals; too easy to pollute nodes silently. The 30s window (vs. 10s elsewhere) is longer because this gate has more density and the user may actually be reviewing it.
- **Step 5 brief-pre-stage gate** → "Wait" after ~10s

The chain should never block. If the user is engaged, gates pause for input. If not, gates pick the conservative default and move on. Skipped Step 2b proposals are logged to the dismissal log per the Step 2b spec so they don't re-surface tomorrow.

---

## Behavior rules

- **Conversational, not formal.** This is a reflection ritual.
- **Skip what doesn't apply.** Missing plugins → skip that step silently.
- **Don't over-capture.** Step 3's cheap-tier triage exists to prevent over-capture; respect its decisions.
- **Honor user gates.** Never auto-convert transcript commitments to CRM tasks without explicit per-item confirmation.
- **Pre-stage is opt-in default.** If `daily-brief` isn't installed, the chain ends after Step 4. No nag.
- **Telemetry (optional).** If core-ops is installed, log one line at completion: `skill: end-day, steps_run: [...], commits_count, runtime_ms`.

## What this command is NOT for

- **Mid-day check-ins.** Use `/recall [node]` or `/search`.
- **Long retrospectives.** Use `/end-week` or `/review`.
- **Session memory dumps.** That's `/remember`. `/end-day` is the *day's* rhythm.
- **Tomorrow's calendar blocking.** That's `plan-tomorrow`'s job (different verb, different output). If you want both this chain AND calendar blocks for tomorrow, run `/end-day` then `/plan-tomorrow` — there's no automatic chain between them in v1.

## When to skip this entirely

- Weekend afternoons where nothing work-shaped happened today
- Days you took off (vacation, sick) — skip the ritual; don't reflect on a non-work-day
- After running `/end-week`, since that subsumes the day-level reflection
