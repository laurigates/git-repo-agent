Synchronize the feature tracker JSON with TODO.md and manage task progress.
## Interaction Mode

Before any closing `report to orchestrator` menu, resolve the automation config:

```bash
bash "${CLAUDE_SKILL_DIR}/../../scripts/get-automation-config.sh"
```

When `EFFECTIVE_INTERACTION_MODE=quiet` **and** this invocation was
automation-initiated (autopilot, session bookend, drift-nudge follow-up — not
a slash command the user typed), skip closing navigation menus ("what next?" /
"create another?" style): apply the safe default and end with a one-line
receipt instead. Quiet mode never skips confirmation gates that guard writes —
only navigation menus. A direct user invocation always behaves fully
interactively (explicit intent overrides quiet; see ADR-0020).


## Mode Selection (run first)

Decide which mode applies before any work:

1. If `--summary` is present, run **Mode: Generate Summary** and exit.
2. If `--drain-wave` is present, run **Mode: Taskwarrior Sidecar Drain** and exit.
3. Otherwise, run sidecar detection (Step 0 below). If a sidecar is detected and
   `TODO.md` is absent, prefer **Sidecar Drain** semantics for any user-facing
   completion prompts; otherwise run **Mode: Full Sync (Default)**.

---


## Mode: Generate Summary (`--summary`)

When `--summary` is provided, generate a human-readable progress report without modifying any files:

```bash
jq -r '
  "# Work Overview: \(.project)\n\n" +
  "## Current Phase: \(.current_phase // "Not set")\n\n" +
  "**Progress**: \(.statistics.complete)/\(.statistics.total_features) features (\(.statistics.completion_percentage)%)\n\n" +
  "### In Progress\n" +
  (if (.tasks.in_progress | length) == 0 then "- (none)\n" else (.tasks.in_progress | map("- \(.description) [\(.id)]") | join("\n")) + "\n" end) +
  "\n### Pending\n" +
  (if (.tasks.pending | length) == 0 then "- (none)\n" else (.tasks.pending | map("- \(.description) [\(.id)]") | join("\n")) + "\n" end) +
  "\n### Recently Completed\n" +
  (if (.tasks.completed | length) == 0 then "- (none)\n" else (.tasks.completed | map("- \(.description) [\(.id)]") | join("\n")) + "\n" end) +
  "\n## Phase Status\n" +
  (.phases | map("- \(.name): \(.status)") | join("\n"))
' docs/blueprint/feature-tracker.json
```

For a sample of the rendered output, see [REFERENCE.md](REFERENCE.md#work-overview-summary-output---summary).

**Exit** after displaying summary.


## Mode: Full Sync (Default)


### Step 0: Run the deterministic core

Run the helper. It owns the mechanical core: taskwarrior-sidecar marker
detection (`SIDECAR=`), tracker existence/validity, the implementation-evidence
backfill (file-existence + `git log` commit dedupe), status inference via the
fixed decision table WITH the never-downgrade guard (`EVIDENCE_FLIPPED=`,
`status_inferred` issues), and the statistics rollup (`STAT_*`,
`COMPLETION_PERCENTAGE=`). It writes the backfilled tracker in place:

```bash
bash "${CLAUDE_SKILL_DIR}/scripts/blueprint-feature-tracker-sync.sh" --home-dir "$HOME" --project-dir "$(pwd)"
```

Parse `STATUS=` and `ISSUES:` from the output. `STATUS=ERROR` means the tracker
is missing (`tracker_missing` → report "Feature tracking not enabled; run
`/blueprint:init`") or invalid JSON. `SIDECAR=true` means the taskwarrior-sidecar
convention is in use — also probe for live taskwarrior linkage (any task with a
`bpid` matching a project blueprint ID) via the parallel-safe `export | jq`
idiom (`task bpid.any: status:any export | jq 'length'`, never `task list`; see
`.claude/rules/parallel-safe-queries.md`). When a sidecar is in play, skip the
`TODO.md` reconciliation steps (Steps 4–5, 8) — there is no authoritative TODO
file — and route any WO closures to **Mode: Taskwarrior Sidecar Drain**.

Each `status_inferred` issue is a feature the evidence flipped up from
`not_started` (the guard never lowers a higher status); surface these under
"Inferred from evidence" in the Step 9 report. For the canonical merge `jq` and
test-evidence patterns, see [REFERENCE.md](REFERENCE.md#evidence-backfill-jq-recipe).


### Step 4: Detect discrepancies

Look for inconsistencies:
- Feature marked `complete` in tracker but unchecked in TODO.md
- Feature checked in TODO.md but not `complete` in tracker
- Feature in `tasks.in_progress` but tracker says `complete`
- PRD status doesn't match feature implementation status
- Feature marked `not_started` but Step 3b inferred shipped code (confirm via Step 5)


### Step 5: Ask user about discrepancies

If discrepancies found (use report to orchestrator):
```
question: "Found {N} discrepancies. How should they be resolved?"
options:
  - label: "Update tracker from TODO.md"
    description: "Trust TODO.md, update tracker to match"
  - label: "Update TODO.md from tracker"
    description: "Trust the tracker, update TODO.md to match"
  - label: "Review each discrepancy"
    description: "Show each discrepancy and decide individually"
  - label: "Skip - don't resolve discrepancies"
    description: "Report discrepancies but don't change anything"
```


### Step 6: Recalculate statistics

The feature-level counts and completion percentage are already in the Step 0
script output (`STAT_COMPLETE`/`STAT_PARTIAL`/`STAT_IN_PROGRESS`/`STAT_NOT_STARTED`/`STAT_BLOCKED`,
`COMPLETION_PERCENTAGE`). After any discrepancy resolutions from Step 5 change a
status, re-derive phase status from the contained features:
  - `complete` if all features complete
  - `in_progress` if any feature in_progress
  - `partial` if some complete, some not
  - `not_started` if no features started


### Step 6a: Resolve portfolio links (v3.3.0+, root blueprints only)

Run only when the manifest at the root has `workspaces.role == "root"` AND the
feature-tracker contains any feature with a non-empty `implemented_by` array.
Skip this step entirely otherwise.

For the child-status rollup table, the `workspaces` summary shape, and the
unresolved-entry warnings, see
[REFERENCE.md](REFERENCE.md#portfolio-link-resolution-v330-root-blueprints-only).


### Step 7: Update feature-tracker.json

- Apply resolved discrepancies
- Update `statistics` section
- Update `last_updated` to today's date
- Update PRD status if features changed
- Update `current_phase` to first incomplete phase


### Step 7a: Verify tracker integrity (deterministic)

Sync **writes** the tracker; this step **verifies** what was written. Run it after
every write path (Full Sync Step 7 and Sidecar Drain Step 5) — nothing else
detects a tracker that drifted by any other route, so the drift compounds
silently:

```bash
bash "${CLAUDE_SKILL_DIR}/../../scripts/blueprint-tracker-check.sh" --project-dir "$(pwd)"
```

Parse `STATUS=` and the `ISSUES:` rows. `statistics` is a **cache** of the
features collection, so `statistics_divergence` rows (which carry
`FIELD=`/`EXPECTED=`/`ACTUAL=`) mean every downstream "N% complete" figure quoted
from this file is wrong — fix the cache in the same run, then re-run the check.

For the per-`TYPE=` response table (`statistics_divergence`,
`feature_status_near_miss`, `feature_status_unknown`,
`task_feature_disagreement`, `fr_cited_not_minted`, `doc_status_stale`,
`dead_statistics_bucket` / `duplicate_timestamp_field`), the manifest
`validation` conventions, and the features-vs-tasks duplicate caveat, see
[REFERENCE.md](REFERENCE.md#tracker-integrity-issue-types).


### Step 8: Update TODO.md (if exists)

- Ensure checkbox states match feature status
- `[x]` for `complete` features
- `[ ]` for `not_started` features
- Note partial completion in task text if needed


### Step 9: Output sync report

Print: statistics block (total/complete/partial/in_progress/not_started/blocked + completion %), current phase, phase-status list, active tasks list, "Changes Made" (status flips, TODO checkboxes touched), "Inferred from evidence" (Step 3b flips with their commit SHAs), and "Unresolved Discrepancies" if any were skipped. See [REFERENCE.md](REFERENCE.md#sync-report-template) for the full report template.


### Step 10: Update task registry

Update the `task_registry["feature-tracker-sync"]` entry in
`docs/blueprint/manifest.json` — `last_completed_at`, `last_result`,
`context.last_todo_hash`, and the `stats` counters. For the `jq` recipe, see
[REFERENCE.md](REFERENCE.md#task-registry-update-jq-recipe).


### Step 11: Prompt for next action

Use report to orchestrator:
```
question: "Sync complete. What would you like to do next?"
options:
  - label: "View detailed status"
    description: "Run /blueprint:feature-tracker-status for full breakdown"
  - label: "Continue development"
    description: "Run /project:continue to work on next task"
  - label: "I'm done"
    description: "Exit sync"
```

---


## Mode: Taskwarrior Sidecar Drain (`--drain-wave`)

Drain one or more completed WOs from `tasks.pending` into `tasks.completed`,
sourcing evidence from taskwarrior annotations (or from named files / an
inline string), then flip any FR-level entries whose implementing WOs are
now all closed.


### Step 1: Parse the wave list

Split `--drain-wave` on commas. For each WO ID, line up the matching evidence
source in this priority order:

1. The matching positional entry in `--evidence-files` (file path), read with
   `jq --rawfile` to dodge single-quote collisions.
2. `--evidence` (single-WO drains only).
3. The latest `annotate` line on the linked taskwarrior task (Step 2).
4. As a last resort, prompt the user for evidence with `report to orchestrator`.

Refuse the run with a clear message if the WO list and `--evidence-files`
list are both provided but their lengths disagree — partial drains are
worse than no drain.


### Step 2: Source evidence from taskwarrior

For each WO in the wave, fetch the latest annotation. Use the parallel-safe
`export | jq` idiom — never `task list` — so a missing-task case returns
exit 0 instead of cancelling sibling tool calls (see
`.claude/rules/parallel-safe-queries.md`):

```bash
task bpid:"$WO" status:completed export \
  | jq -r '.[0].annotations | sort_by(.entry) | last | .description // empty'
```

If the result is empty, fall back to `status:any` (the user may have closed
the task before drain). If still empty, fall back to the next priority source
from Step 1.

Persist each evidence string to a temp file (`mktemp`) — embedded single
quotes in commit messages collide with shell when inlined into a `jq`
program literal, and `--rawfile` is the standard escape:

```bash
ev_file="$(mktemp)"
printf '%s' "$EVIDENCE_STRING" > "$ev_file"
```


### Step 3: Drain pending → completed

For each `WO-NNN` in the wave, with its evidence file `$ev_file`, advance
the tracker in a single `jq` pass per WO. Store the date once and pass it
in as an argument so the same value lands on every entry:

```bash
today="$(date -u +%Y-%m-%d)"
jq --arg id "$WO" \
   --arg today "$today" \
   --rawfile ev "$ev_file" '
  .tasks.completed = (
    [ .tasks.pending[]
      | select(.id == $id)
      | . + {"completed": $today, "evidence": $ev}
    ] + .tasks.completed
  )
  | .tasks.pending = [.tasks.pending[] | select(.id != $id)]
' docs/blueprint/feature-tracker.json > docs/blueprint/feature-tracker.json.tmp
mv docs/blueprint/feature-tracker.json.tmp docs/blueprint/feature-tracker.json
```

Loop the WOs sequentially — each pass reads the file the previous pass
wrote — so concurrent writes cannot collide on the same file.

If a WO ID is not in `tasks.pending`, report `skipped: not pending` for
that entry and continue. Do not error the whole wave.


### Step 4: Flip FR status when implementing WOs are all closed

For each feature whose `implementing_wos` array overlaps the drained wave,
recompute its `status`. The flip is the second hand-jq pattern users
repeat per wave; do it once here. For the `jq` recipe, see
[REFERENCE.md](REFERENCE.md#fr-status-flip-jq-recipe---drain-wave-step-4).

If the tracker schema stores features in a flat `features` array but with a
different shape (e.g., nested under `phases[].features[]`), adapt the path
prefix while preserving the same logic: a feature flips to `complete` only
when **every** WO ID listed in `implementing_wos` appears in
`tasks.completed`.

Record each flip in the run report (Step 6). Never silently downgrade an
already-`complete` FR.


### Step 5: Recalculate statistics

Re-run Step 6 of **Mode: Full Sync (Default)** so the totals reflect the
drained WOs and any flipped FRs. Then write the updated `last_updated` and
`current_phase` per Step 7 of Full Sync, and verify the result with **Step 7a**
of Full Sync — a drain moves ids between `tasks.pending` and `tasks.completed`,
which is exactly when `statistics` and task/feature agreement drift.


### Step 6: Report

Print a Drain Report covering the wave list, each WO's drained/skipped outcome
with its evidence source, the FR flips, the updated statistics, and the
`/taskwarrior:task-done` follow-up. For the report template, see
[REFERENCE.md](REFERENCE.md#sidecar-drain-report-example).

Clean up temp evidence files with `rm -f "$ev_file"`.


### Single-WO short form

For the common one-WO case, the same flow with `--drain-wave WO-031` and
either `--evidence "<text>"` or no evidence flag (annotation autosourced) is
shorter than the legacy hand-rolled `jq` one-liner — and emits the same
on-disk shape. Prefer `/taskwarrior:task-done` when you also need to close
the linked taskwarrior task; this skill only edits the tracker.

---


## Direct Edits, Recipes & Sample Output

For ad-hoc tracker surgery (`jq` recipes for adding to `in_progress`, completing tasks, queueing pending work), the evidence-backfill / task-registry / FR-status-flip `jq` recipes, the portfolio-link rollup rules, the tracker-integrity issue-type responses, and the sync / summary / drain report samples, see .


## Related

- `taskwarrior-plugin:task-done` — close a single taskwarrior task and drain
  the linked tracker entry; pairs with this skill's `--drain-wave` for
  wave-granular drains where multiple WOs land at once.
- `taskwarrior-plugin:task-coordinate` — surface the next N unblocked tasks
  before starting a wave, so the WOs you eventually drain here line up with
  what the queue actually scheduled.
- `session-plugin:session-end` — the session wind-down orchestrator offers a
  `--drain-wave` pass when its survey finds closed WO-linked (`bpid`)
  taskwarrior tasks still sitting in the tracker's `tasks.pending`, so the
  drain happens at the session bookend instead of drifting until someone
  remembers to run this sync by hand.
- `.claude/rules/parallel-safe-queries.md` — the `task ... export | jq`
  idiom is mandatory whenever this skill queries taskwarrior. `task list`
  exits 1 on empty results and silently cancels sibling parallel tool calls.


# Feature Tracker Sync — Reference

Reference material for `blueprint-feature-tracker-sync`: direct-edit `jq` recipes for tracker mutations, the portfolio-link rollup rules, the tracker-integrity issue-type responses, and sample report output. The execution flow itself lives in [SKILL.md](SKILL.md).


## Direct Tracker Edits (jq Recipes)

These recipes manipulate `docs/blueprint/feature-tracker.json` directly. Prefer the skill's mode-driven flows (`--summary`, `--drain-wave`, default full sync) for routine work — these recipes are for ad-hoc surgery.


### Adding a task to in_progress

When starting work on a feature:

```bash
jq '.tasks.in_progress += [{"id": "FR2.3", "description": "Implement OAuth integration", "source": "PRP-002", "added": "2026-02-04"}]' \
  docs/blueprint/feature-tracker.json > tmp.json && mv tmp.json docs/blueprint/feature-tracker.json
```


### Completing a task

When finishing work:

```bash

# Move from in_progress to completed (keep last 10)
jq '
  .tasks.completed = ([.tasks.in_progress[] | select(.id == "FR2.3") | . + {"completed": "2026-02-04"}] + .tasks.completed)[:10] |
  .tasks.in_progress = [.tasks.in_progress[] | select(.id != "FR2.3")]
' docs/blueprint/feature-tracker.json > tmp.json && mv tmp.json docs/blueprint/feature-tracker.json
```


### Adding pending tasks

When planning future work:

```bash
jq '.tasks.pending += [{"id": "FR4.1", "description": "Webhook support", "source": "PRD-001", "added": "2026-02-04"}]' \
  docs/blueprint/feature-tracker.json > tmp.json && mv tmp.json docs/blueprint/feature-tracker.json
```


## Evidence Backfill jq Recipe

After Step 3b scans the working tree and git history, merge results into the tracker. For each feature `$FR_ID` with scanned `$NEW_COMMITS` (newline-separated SHAs in `/tmp/scan-commits.txt`), `$NEW_TESTS` (newline-separated paths in `/tmp/scan-tests.txt`), and an `$INFERRED_STATUS` of `complete` / `partial` / `null`:

```bash
jq --arg id "$FR_ID" \
   --rawfile commits /tmp/scan-commits.txt \
   --rawfile tests /tmp/scan-tests.txt \
   --arg status "$INFERRED_STATUS" \
   --arg today "$(date -u +%Y-%m-%d)" '
  (.features // []) |= map(
    if .id == $id then
      . as $fr
      | .implementation.commits = (
          ((.implementation.commits // []) +
           ($commits | split("\n") | map(select(length > 0))))
          | unique
        )
      | .implementation.tests = (
          ((.implementation.tests // []) +
           ($tests | split("\n") | map(select(length > 0))))
          | unique
        )
      | if ($fr.status // "not_started") == "not_started" and $status != "null"
        then .status = $status
             | (if $status == "complete" then .completed_at = $today else . end)
        else .
        end
    else .
    end
  )
' docs/blueprint/feature-tracker.json > docs/blueprint/feature-tracker.json.tmp
mv docs/blueprint/feature-tracker.json.tmp docs/blueprint/feature-tracker.json
```

Run sequentially per feature so concurrent writes don't collide. The recipe preserves any existing commit/test entries (deduped via `unique`) and only flips `status` upward from `not_started` — already-`complete`/`in_progress`/`partial` features are left alone.


## Task Registry Update jq Recipe

Full Sync Step 10 updates the task registry entry in `docs/blueprint/manifest.json`:

```bash
jq --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --arg todo_hash "$(sha256sum TODO.md 2>/dev/null | cut -d' ' -f1)" \
  --argjson processed "${FEATURES_SYNCED:-0}" \
  '.task_registry["feature-tracker-sync"].last_completed_at = $now |
   .task_registry["feature-tracker-sync"].last_result = "success" |
   .task_registry["feature-tracker-sync"].context.last_todo_hash = $todo_hash |
   .task_registry["feature-tracker-sync"].stats.runs_total = ((.task_registry["feature-tracker-sync"].stats.runs_total // 0) + 1) |
   .task_registry["feature-tracker-sync"].stats.items_processed = $processed' \
  docs/blueprint/manifest.json > tmp.json && mv tmp.json docs/blueprint/manifest.json
```


## FR Status Flip jq Recipe (`--drain-wave` Step 4)

For each feature whose `implementing_wos` array overlaps the drained wave, recompute its `status`:

```bash
jq --arg today "$today" '
  (.features // [])
  |= map(
    if (.implementing_wos // []) | length > 0 then
      . as $fr
      | (.implementing_wos
         | map(. as $woid
               | (($fr | .. | objects | select(has("id")) | select(.id == $woid))
                  // null)
               | . != null)) as $resolved
      | (((.implementing_wos | length) > 0)
         and ([.implementing_wos[] as $wo
                | any(($fr.parent_tracker.tasks.completed // [])[]; .id == $wo)]
              | all)) as $all_done
      | if $all_done and (.status // "") != "complete"
        then . + {"status": "complete", "completed_at": $today}
        else .
        end
    else .
    end
  )
' docs/blueprint/feature-tracker.json > docs/blueprint/feature-tracker.json.tmp
mv docs/blueprint/feature-tracker.json.tmp docs/blueprint/feature-tracker.json
```


## Portfolio Link Resolution (v3.3.0+, root blueprints only)

Full Sync Step 6a. Runs only when the manifest at the root has `workspaces.role == "root"` AND the feature-tracker contains any feature with a non-empty `implemented_by` array.

1. For each feature with `implemented_by`:
   - For every `{workspace, ref}` entry, read
     `<workspace>/docs/blueprint/feature-tracker.json` and look up `ref`.
   - Collect the child statuses. If any entry cannot be resolved (missing file
     or missing ref), record a warning and treat that entry as `not_started`
     for the rollup.
   - Derive the root feature's `status` using this rule:

     | Child statuses observed | Derived status |
     |-------------------------|----------------|
     | All resolved entries `complete` | `complete` |
     | Any `blocked` | `blocked` |
     | Any `in_progress`, or a mix of `complete`/`not_started` | `partial` |
     | All `not_started` | `not_started` |

   - Overwrite the feature's `status` with the derived value. Do NOT touch
     `implementation` on portfolio features; status alone is recomputed.

2. Rebuild the top-level `workspaces` summary by reading each child's
   `statistics` block:

   ```json
   "workspaces": {
     "projects/esp32-lamp": {
       "total": 14, "complete": 6, "completion_percentage": 42.9,
       "current_phase": "phase-1", "last_synced_at": "<now>"
     }
   }
   ```

3. Recompute root `statistics` after the derived statuses are applied so the
   portfolio-level totals reflect the child-driven states.

4. Emit warnings in the sync report (Step 9) for unresolved `implemented_by`
   entries, and suggest `/blueprint:workspace-scan` when a referenced
   workspace is not present in the root manifest's `workspaces.children`.


## Tracker Integrity Issue Types

Responses to the `ISSUES:` rows emitted by `blueprint-tracker-check.sh` (Full Sync Step 7a):

| `TYPE=` | Response |
|---------|----------|
| `statistics_divergence` | Write the `EXPECTED=` value; the checker uses the same `round(complete/total*1000)/10` formula as Step 6 |
| `feature_status_near_miss` | Rewrite the status to the `CANONICAL=` spelling |
| `feature_status_unknown` | report to the orchestrator which schema status was meant (`not_started`, `in_progress`, `partial`, `complete`, `blocked`) |
| `task_feature_disagreement` | An undrained `tasks.pending` entry (route to `--drain-wave`) or a task closed ahead of its feature |
| `fr_cited_not_minted` | Mint the FR, or correct the citing document — an unminted FR is invisible to every status query |
| `doc_status_stale` | Offer to advance the doc's frontmatter `status:` (it is still unfinished while every FR it cites has landed) |
| `dead_statistics_bucket` / `duplicate_timestamp_field` | Drop the non-schema bucket; keep `last_updated`, drop the alias |

Repo conventions (document status vocabulary, doc globs, excluded basenames)
come from the manifest `validation` block via
`scripts/get-validation-config.sh` — absent, it uses working defaults.

The check never reports an FR id appearing in **both** the features collection
and a task list as a duplicate: that repetition is the documented drain design.
Status is read from the features collection only.


## Sync Report Template

```
Feature Tracker Sync Report
===========================
Last Updated: {date}

Statistics:
- Total Features: {total}
- Complete: {complete} ({percentage}%)
- Partial: {partial}
- In Progress: {in_progress}
- Not Started: {not_started}
- Blocked: {blocked}

Current Phase: {current_phase}

Phase Status:
- Phase 0: {status}
- Phase 1: {status}
...

Active Tasks:
{tasks.in_progress | list}

Changes Made:
{If changes made:}
- {feature}: {old_status} -> {new_status}
- Updated TODO.md: checked {N} items
{If no changes:}
- No changes needed, all in sync

Inferred from evidence (Step 3b):
{For each feature flipped from not_started:}
- {feature_id} ({feature_title}): not_started -> {inferred_status}
  Files: {implementation.files | join(", ")}
  Commits backfilled: {N} SHAs

{If discrepancies skipped:}
Unresolved Discrepancies:
- {feature}: tracker says {status}, TODO.md shows {checkbox_state}
```


## Example Summary Output

```
Feature Tracker Sync Report
===========================
Last Updated: 2026-02-04

Statistics:
- Total Features: 42
- Complete: 22 (52.4%)
- Partial: 4
- In Progress: 2
- Not Started: 14
- Blocked: 0

Current Phase: phase-2

Phase Status:
- Phase 0: complete
- Phase 1: complete
- Phase 2: in_progress
- Phase 3-8: not_started

Active Tasks:
- Implement OAuth integration [FR2.3]
- Add rate limiting [FR3.1]

Changes Made:
- FR2.6.1 (Skill Progression): partial -> complete
- FR2.6.2 (Experience Points): not_started -> complete
- Updated TODO.md: checked 2 items

All sync targets updated successfully.
```


## Work Overview Summary Output (`--summary`)

Output example:
```markdown

# Work Overview: my-project


## Current Phase: phase-1

**Progress**: 22/42 features (52.4%)


### In Progress
- Implement OAuth integration [FR2.3]
- Add rate limiting [FR3.1]


### Pending
- Webhook support [FR4.1]
- Admin dashboard [FR5.1]


### Recently Completed
- User authentication [FR2.1]
- Session management [FR2.2]


## Phase Status
- Foundation: complete
- Core Features: in_progress
- Advanced Features: not_started
```


## Sidecar Drain Report Example

Printed by `--drain-wave` Step 6:

```
Sidecar Drain Report
====================
Wave: WO-031, WO-032, WO-033
Drained:
- WO-031: pending -> completed  (evidence: 142 chars from tw annotation)
- WO-032: pending -> completed  (evidence: 209 chars from /tmp/wo032_ev.txt)
- WO-033: skipped (not in tasks.pending)

FR flips:
- FR-017 (Skill Progression): in_progress -> complete

Statistics:
- Total Features: 42
- Complete: 23 (54.8%)  [+1 from FR-017]
- Recently Completed: WO-031, WO-032 added to top of tasks.completed

Next: run /taskwarrior:task-done if any sibling tasks should also close.
```