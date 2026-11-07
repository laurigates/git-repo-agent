Upgrade the blueprint structure to the latest format version.
## Execution

Execute this upgrade workflow. Step numbers are stable: the reference files and the non-interactive defaults cite them.


### Step 1: Check current state

Run the snippet in Step 2. It resolves `$MANIFEST` (if no manifest exists, suggest `/blueprint:init` instead) and reads the current `format_version`, defaulting to "1.0.0" when the field is missing.


### Step 2: Determine upgrade path

```bash

# Resolve manifest path once — use $MANIFEST in all subsequent jq commands.

# Same order as blueprint-plugin/scripts/get-validation-config.sh.
if [[ -f docs/blueprint/manifest.json ]]; then
  MANIFEST=docs/blueprint/manifest.json        # v3.0+ canonical (what /blueprint:init writes)
elif [[ -f docs/blueprint/.manifest.json ]]; then
  MANIFEST=docs/blueprint/.manifest.json       # v3.0 dot-prefixed variant from early migrations
elif [[ -f .claude/blueprints/.manifest.json ]]; then  # v1.x/v2.x location
  MANIFEST=.claude/blueprints/.manifest.json
else
  echo "ERROR: no blueprint manifest found. Run /blueprint:init first."
  exit 1
fi
current=$(jq -r '.format_version // "1.0.0"' "$MANIFEST")
target="3.4.0"
```

**Important**: Use the resolved `$MANIFEST` in every `jq` call here and in delegated migration steps — a hard-coded filename fails silently.

**Version compatibility matrix**:

| From Version | To Version | Migration Document |
|--------------|------------|-------------------|
| 1.0.x        | 1.1.x      | `migrations/v1.0-to-v1.1.md` |
| 1.x.x        | 2.0.0      | `migrations/v1.x-to-v2.0.md` |
| 2.x.x        | 3.0.0      | `migrations/v2.x-to-v3.0.md` |
| 3.0.x        | 3.1.0      | `migrations/v3.0-to-v3.1.md` |
| 3.1.x        | 3.2.0      | inline (Step 3a) |
| 3.2.x        | 3.3.0      | `migrations/v3.2-to-v3.3.md` |
| 3.3.x        | 3.4.0      | `migrations/v3.3-to-v3.4.md` |
| 3.4.0        | 3.4.0      | Already up to date |


### Step 3: Check for deprecated generated commands

Detect skills/commands left by the deprecated `/blueprint:generate-commands` and offer to remove them (auto-"Yes" when `$NONINTERACTIVE`) per [references/deprecated-commands.md](references/deprecated-commands.md). If none are found, continue to Step 4.


### Step 3a: v3.1 → v3.2 migration — add task registry

If `task_registry` is missing: ask the scheduling question ("Prompt before running" when `$NONINTERACTIVE`), add the registry, and bump `format_version` to 3.2.0 per [references/v3.1-to-v3.2-task-registry.md](references/v3.1-to-v3.2-task-registry.md).


### Step 3b: v3.2 → v3.3 migration — monorepo support

Delegate to `skills/blueprint-migration/migrations/v3.2-to-v3.3.md` (summary: [references/v3.2-to-v3.3-summary.md](references/v3.2-to-v3.3-summary.md)).


### Step 3c: v3.3 → v3.4 migration — automation block (autonomy levels)

Delegate to `skills/blueprint-migration/migrations/v3.3-to-v3.4.md` (summary: [references/v3.3-to-v3.4-summary.md](references/v3.3-to-v3.4-summary.md)). In `$NONINTERACTIVE` mode take the suggested `autonomy_level`, never higher.


### Step 3d: Re-key `generated.rules` lacking `.md` (any version, #2868)

Dry-run only; nothing is written before Step 5:

```bash
bash "${CLAUDE_SKILL_DIR}/../../scripts/migrate-generated-rules-keys.sh" --manifest "$MANIFEST" --dry-run
```

List its `ISSUES:` in the Step 4 plan; Step 6 applies it.


### Step 4: Display upgrade plan

Print the upgrade plan in [references/upgrade-plan.md](references/upgrade-plan.md).


### Step 5: Confirm with user

If `$NONINTERACTIVE` is `true`, skip this confirmation and proceed directly to Step 6.

Otherwise, use report to orchestrator:

```
question: "Ready to upgrade blueprint from v{current} to v3.4.0?"
options:
  - "Yes, upgrade now" → proceed
  - "Show detailed migration steps" → display migration document
  - "Create backup first" → run git stash or backup then proceed
  - "Cancel" → exit
```


### Step 6: Load and execute migration document

- Read the appropriate migration document from `blueprint-migration` skill
- For v1.x → v2.0: Load `migrations/v1.x-to-v2.0.md` (overview: Step 7)
- For v2.x → v3.0: Load `migrations/v2.x-to-v3.0.md` (overview: Step 8)
- For v3.0 → v3.1: Load `migrations/v3.0-to-v3.1.md`
- For v3.1 → v3.2: Execute Step 3a
- For v3.2 → v3.3: Load `migrations/v3.2-to-v3.3.md` (see Step 3b summary)
- For v3.3 → v3.4: Load `migrations/v3.3-to-v3.4.md` (see Step 3c summary)
- If Step 3d reported findings: re-run its command without `--dry-run`; report `MIGRATED=` and `ISSUES:` in Step 10
- Execute each step with user confirmation for destructive operations


### Step 7: v1.x → v2.0 migration overview (from migration document)

Sub-steps a–g, including the 7f/7g prompts and their `$NONINTERACTIVE` defaults, are in [references/v1.x-to-v2.0-overview.md](references/v1.x-to-v2.0-overview.md).


### Step 8: v2.x → v3.0 migration overview (from migration document)

Sub-steps a–f are in [references/v2.x-to-v3.0-overview.md](references/v2.x-to-v3.0-overview.md).


### Step 9: Update manifest (v3.0.0 schema)

Write the manifest using the v3.0.0 schema template in . Preserve `created_at`, `project.name`, `project.type`, `structure.has_modular_rules`, and `structure.claude_md_mode` from the previous manifest. Set `updated_at` to now, `created_by.blueprint_plugin` to the plugin version (`jq -r .version "${CLAUDE_SKILL_DIR}/../../.claude-plugin/plugin.json"`, never the format version, issue #2950), and `generated.rules` from the migrated skills→rules conversion.


### Step 10: Report

Print the upgrade report using the standard template in , substituting `{previous}` with the prior `format_version` and `{n}` placeholders with the actual counts.


### Step 11: Prompt for next action

If `$NONINTERACTIVE` is `true`, skip this prompt entirely — print the report from Step 10 and return. The batch caller owns follow-up (status checks, commits, etc.).

Otherwise, ask with the prompt in [references/next-action-prompt.md](references/next-action-prompt.md), and act on the selection as that file maps it.


### After any version bump: verify, or roll back

**Post-migration assertion**: after any version bump, run the `format_version` check in [REFERENCE.md](REFERENCE.md#post-migration-assertion). **Rollback**: if the upgrade fails, follow [REFERENCE.md](REFERENCE.md#rollback).


# Blueprint Upgrade — Reference

Reference templates for `/blueprint:upgrade`: the v3.0.0 manifest schema produced by step 9, the standard upgrade report template emitted by step 10, the post-migration assertion, and the rollback procedure.


## v3.0.0 Manifest Schema

```json
{
  "format_version": "3.0.0",
  "created_at": "[preserved]",
  "updated_at": "[now]",
  "created_by": {
    "blueprint_plugin": "[plugin version, read from plugin.json]"
  },
  "project": {
    "name": "[preserved]",
    "type": "[preserved]",
    "detected_stack": []
  },
  "structure": {
    "has_prds": true,
    "has_adrs": "[detected]",
    "has_prps": "[detected]",
    "has_work_orders": true,
    "has_ai_docs": "[detected]",
    "has_modular_rules": "[preserved]",
    "has_document_detection": "[based on user choice]",
    "claude_md_mode": "[preserved]"
  },
  "generated": {
    "rules": {
      "[rule-name]": {
        "source": "docs/prds/...",
        "source_hash": "sha256:...",
        "generated_at": "[now]",
        "plugin_version": "3.0.0",
        "content_hash": "sha256:...",
        "status": "current"
      }
    },
    "commands": {}
  },
  "task_registry": {
    "// note": "Added by v3.1 → v3.2 migration step above"
  },
  "custom_overrides": {
    "rules": ["[any promoted rules]"],
    "commands": []
  },
  "upgrade_history": [
    {
      "from": "{previous}",
      "to": "3.0.0",
      "date": "[now]",
      "changes": ["Moved state to docs/blueprint/", "Converted skills to rules", "..."]
    }
  ]
}
```


## Step 10 Report Template

```
Blueprint upgraded successfully!

v{previous} → v3.0.0

State files moved to docs/blueprint/:
- .manifest.json
- feature-tracker.json
- work-orders/ directory
- ai_docs/ directory

Generated rules (.claude/rules/):
- {n} rules (converted from skills)

Custom layer (.claude/skills/):
- {n} promoted rules (preserved modifications)
- {n} promoted skills

[Document detection: enabled (if selected)]

Task registry:
- {n} tasks registered with scheduling metadata
- Auto-run mode: {user choice from migration step}
- Run /blueprint:status to see task health dashboard

New v3.0 architecture:
- Blueprint state: docs/blueprint/ (version-controlled with project)
- Generated rules: .claude/rules/ (project-specific context)
- Custom layer: Your overrides, never auto-modified
- Removed: .claude/blueprints/generated/ (no longer needed)
```


## Post-Migration Assertion

After any version bump, verify `format_version` actually changed to the target. This catches silent failures where `jq` operated on the wrong path and exited 0 with empty output:

```bash
actual=$(jq -r '.format_version' "$MANIFEST")
if [[ "$actual" != "$target" ]]; then
  echo "ERROR: Migration failed — format_version is '$actual', expected '$target'"
  echo "Check that $MANIFEST was written correctly and rerun the migration step."
  exit 1
fi
echo "Migration verified: format_version = $actual in $MANIFEST"
```


## Rollback

If upgrade fails:
- Check git status for changes made
- Use `git checkout -- .claude/` and `git checkout -- docs/blueprint/` to restore original structure
- Manually move content back if needed
- Report specific failure point for debugging