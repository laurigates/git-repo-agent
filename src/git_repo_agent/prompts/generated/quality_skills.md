## code-review-checklist

# Code Review Checklist

Structured approach to reviewing code changes.


## Review Priority Order

1. **Security** (Critical) - Vulnerabilities, secrets, injection
2. **Correctness** (High) - Logic errors, breaking changes
3. **Performance** (Medium) - Inefficiencies, resource leaks
4. **Quality** (Medium) - Maintainability, readability
5. **Style** (Low) - Formatting, naming (should be automated)


## Security Checklist


### Secrets & Credentials
- [ ] No hardcoded API keys, passwords, tokens
- [ ] No credentials in logs or error messages
- [ ] Secrets loaded from environment/vault


### Injection Vulnerabilities
- [ ] SQL queries use parameterized statements
- [ ] User input is sanitized before HTML output (XSS)
- [ ] Shell commands don't include user input (command injection)
- [ ] File paths are validated (path traversal)


### Authentication & Authorization
- [ ] Auth checks on all protected endpoints
- [ ] Proper session handling
- [ ] Secure password handling (hashing, not plaintext)


### Data Exposure
- [ ] Sensitive data not logged
- [ ] API responses don't leak internal details
- [ ] Error messages don't expose system info


## Correctness Checklist


### Logic
- [ ] Edge cases handled (null, empty, boundary values)
- [ ] Error conditions handled appropriately
- [ ] Async operations properly awaited
- [ ] Race conditions considered


### Breaking Changes
- [ ] API contracts maintained
- [ ] Database migrations are reversible
- [ ] Feature flags for risky changes


### Testing
- [ ] New code has tests
- [ ] Tests cover error paths, not just happy path
- [ ] Existing tests still pass


## Performance Checklist


### Efficiency
- [ ] No N+1 queries
- [ ] Appropriate data structures used
- [ ] No unnecessary loops or iterations
- [ ] Caching considered for expensive operations


### Resources
- [ ] Database connections closed/pooled
- [ ] File handles closed
- [ ] No memory leaks (event listeners removed, etc.)


### Scale
- [ ] Works with realistic data volumes
- [ ] Pagination for large result sets
- [ ] Timeouts on external calls


## Quality Checklist


### Readability
- [ ] Clear, descriptive names
- [ ] Functions do one thing
- [ ] No overly complex conditionals
- [ ] Comments explain "why", not "what"


### Maintainability
- [ ] DRY (no copy-paste duplication)
- [ ] Appropriate abstractions
- [ ] Dependencies are justified
- [ ] No dead code


### Consistency
- [ ] Follows project patterns
- [ ] Matches existing code style
- [ ] Uses established utilities/helpers


## Review Output Format

```markdown

## Review: [PR Title]

**Risk Level**: LOW | MEDIUM | HIGH | CRITICAL


### Critical Issues
1. [Category] Description (file:line)
   - Impact: What could go wrong
   - Fix: Specific recommendation


### Suggestions
1. [Category] Description (file:line)
   - Why: Reasoning
   - Consider: Alternative approach


### Positive Notes
- [Recognition of good patterns]
```


## Quick Checks

For fast reviews, at minimum check:
1. Any secrets or credentials?
2. Any SQL/command injection?
3. Are error cases handled?
4. Do tests exist for new code?

---

## code-lint

## Execution

Run this lint pass:


### Step 0: One-shot path (preferred when no `PATH` is given)

When the caller passed no path (or passed `.`), the bundled detector already does
detection, tool discovery, and fixing in a single call:

```bash
bash "${CLAUDE_PLUGIN_ROOT}/skills/code-lint/scripts/detect-and-fix.sh"
bash "${CLAUDE_PLUGIN_ROOT}/skills/code-lint/scripts/detect-and-fix.sh" --check-only
```

Pass `--check-only` unless `FIX` is set. It detects biome, eslint, prettier,
ruff, black, clippy, rustfmt, gofmt, golangci-lint, and shellcheck, reports which
were found, and shows modified files. Report its output and stop.

Continue to Step 1 when the caller scoped the run to a specific `PATH`, or when
the detector reports no linters found.


### Step 1: Detect the project language

Read the `Package files` line from Context above (or `ls` the target directory)
and map each marker file to a language. **The signals are the marker files — do
not guess from file extensions or the repo name.**

| Marker file present in the repo root | Language |
|---|---|
| `pyproject.toml`, `setup.py`, `requirements.txt` | Python |
| `package.json` | JavaScript / TypeScript |
| `Cargo.toml` | Rust |
| `go.mod` | Go |

- **Exactly one match** → run that row of Step 2.
- **Several matches** (polyglot repo) → run each matching row, then aggregate the
  results into one summary. Do not pick one arbitrarily.
- **No match** → skip to Step 3.


### Step 2: Run the row for each detected language

Run the **Lint** and **Format** commands for every detected language. Swap in the
`--fix` / `--format` variants only when the caller set those flags.

| Language | Lint (always) | Lint with `--fix` | Format check (default) | Format with `--format` |
|---|---|---|---|---|
| Python | `uv run ruff check PATH --output-format=concise` | `uv run ruff check PATH --output-format=concise --fix` | `uv run ruff format --check PATH` | `uv run ruff format PATH` |
| JavaScript / TypeScript | `npx eslint PATH` | `npx eslint PATH --fix` | `npx prettier --check PATH` | `npx prettier --write PATH` |
| Rust | `cargo clippy --message-format=short -- -D warnings` | `cargo clippy --fix --allow-dirty` | `cargo fmt -- --check` | `cargo fmt` |
| Go | `go vet ./...` | (no autofix — fix by hand) | `gofmt -l PATH` | `gofmt -w PATH` |

Then run each detected language's extra checks:

| Language | Extra checks |
|---|---|
| Python | `uv run ty check PATH --hide-progress`, `uv run bandit -q -r PATH` |
| JavaScript / TypeScript | `npx tsc --noEmit` |
| Rust | `cargo check` |
| Go | `staticcheck ./...` (only if installed) |

When the repo defines its own lint script (`npm run lint`, a `just lint` recipe),
prefer it over the raw command above — it encodes the project's own flags.


### Step 3: Fallback when no language was detected

In order, stopping at the first that applies:

1. `Makefile` present → `make lint`
2. `package.json` with a `lint` script → `npm run lint`
3. Otherwise report that no linters were found, and suggest `/deps:install --dev`
   and `/configure:linting`.


### Step 4: Pre-commit integration

If the `Pre-commit config` line in Context is non-empty:

| Caller flags | Command |
|---|---|
| default | `pre-commit run --all-files` |
| `--fix` | `pre-commit run --all-files --show-diff-on-failure` |


## Autofix Command Reference

| Language | Linter | Autofix Command |
|----------|--------|-----------------|
| TypeScript/JS | biome | `npx @biomejs/biome check --write .` |
| TypeScript/JS | biome format | `npx @biomejs/biome format --write .` |
| Python | ruff | `ruff check --fix .` |
| Python | ruff format | `ruff format .` |
| Rust | clippy | `cargo clippy --fix --allow-dirty` |
| Rust | rustfmt | `cargo fmt` |
| Go | gofmt | `gofmt -w .` |
| Go | go mod | `go mod tidy` |
| Shell | shellcheck | No autofix (manual only) |


### Common Fix Patterns

**JavaScript/TypeScript (Biome)**: unused imports, prefer-const (`let x = 5` → `const x = 5`).

**Python (Ruff)**: import sorting (I001), unused imports (F401), long lines auto-wrapped.

**Rust (Clippy)**: redundant clone, `match` → `if let` for single-arm patterns.

**Shell (ShellCheck — manual fixes)**: quote variables (`$var` → `"$var"`), use `$()` instead of backticks.


### When to Escalate from Autofix

Stop autofix and use a different approach when:
- Fix requires understanding business logic
- Multiple files need coordinated changes
- Warning indicates a potential bug (not just style)
- Security-related linter rule
- Type error requires interface/API changes


## Post-lint Actions

After linting:
1. Summary of issues found/fixed
2. If unfixable issues exist, suggest `/code:refactor` command
3. If all clean, ready for `/git:smartcommit`

---

## dry-consolidation

# DRY Consolidation

Systematic extraction of duplicated code into shared, tested abstractions.


## Execution

Execute this 7-step consolidation workflow. Use TodoWrite to track each extraction as a separate task.


### Step 1: Discover duplicate clusters (deterministic clone detection)

Enumerate duplicate ranges with a deterministic clone detector, then read **only the reported ranges** — not whole candidate files. This keeps discovery reproducible and cheap. Token-based detection (jscpd) finds copy-paste independent of whitespace/formatting and of the enclosing symbol name — clone pairs a name-based Grep misses when the wrapping function is renamed. ast-grep (1b) then adds tolerance for variables renamed *inside* the block.

#### 1a. Token-based near-duplicates with jscpd

`jscpd` is a token-based copy/paste detector that supports 150+ languages despite the "js" in the name; `npx` runs it with no global install. Run it over the target path:

```bash
npx jscpd --reporters json --min-tokens 50 --output /tmp/jscpd-dry --silent <path>
```

It writes `/tmp/jscpd-dry/jscpd-report.json`. Read that report and parse its `duplicates` array — each entry gives the exact file/line ranges of a clone pair plus its size in tokens/lines:

```json
{
  "duplicates": [
    {
      "format": "tsx",
      "lines": 12,
      "tokens": 84,
      "firstFile":  { "name": "src/UserList.tsx",  "start": 20, "end": 32 },
      "secondFile": { "name": "src/OrderList.tsx", "start": 15, "end": 27 }
    }
  ],
  "statistics": { "total": { "clones": 3, "duplicatedLines": 40, "duplicatedTokens": 252, "percentage": 5.1 } }
}
```

For each reported clone, **Read only the line ranges** (`Read` with `offset`/`limit` around `start`/`end`) to confirm the duplication and classify it — do not Read whole candidate files. jscpd similarity is high by construction for a reported clone (a `--min-tokens` match); note the tokens/lines for the Extraction Plan.

#### 1b. Structural confirmation with ast-grep

Once jscpd surfaces a cluster, confirm it is the same *shape* — same call-shape / same block modulo captured variables — with ast-grep metavariables. `$VAR` / `$INIT` match any identifier/expression, so a block differing only in renamed captures still matches:

```bash
ast-grep -p 'const $VAR = useState($INIT)' --lang tsx <path>
```

Use this to separate a genuine extractable duplicate from a coincidental token overlap before planning the extraction. (For a standalone structural search without extraction, use the `ast-grep-search` skill.)

#### 1c. Graceful fallback (Grep) when the detector is unavailable

When `npx`/`jscpd` is unavailable, or the ecosystem has no `npx` on PATH, fall back to agent-driven text search:

1. Use Grep to find repeated function names, variable patterns, and import clusters
2. Use Glob to identify files with similar structure (e.g., all `*List.tsx`, all `*Detail.tsx`)
3. Read candidate files to confirm duplication and measure scope

This fallback has lower recall for near-duplicates (renamed variables, reordered params) — prefer the jscpd path when available, and reserve Grep for when it is not.

**Duplication signals to classify** (both the jscpd and the Grep path feed the same categories in Step 2):
- Utility functions defined identically in multiple files (string truncation, date formatting, validation)
- Identical error handling blocks (try/catch patterns, error state JSX)
- Copy-pasted UI fragments (pagination controls, confirmation dialogs, loading states)
- Repeated hook/state management patterns (delete confirmation + mutation + handler)
- Duplicated import blocks that signal repeated inline implementations


### Step 2: Classify duplications

Group discovered duplications into extraction categories:

| Category | Extract Into | Location Convention |
|----------|-------------|---------------------|
| **Utilities** | Pure functions | `src/lib/utils/` or `src/utils/` |
| **Components** | Shared UI components | `src/components/ui/` or `src/components/shared/` |
| **Hooks** | Custom React/Vue hooks | `src/hooks/` or `src/composables/` |
| **Types** | Shared type definitions | `src/types/` or alongside the abstraction |

Follow the project's existing conventions for shared code location. If no convention exists, propose one based on the framework.


### Step 3: Plan extractions

For each duplication cluster, plan the extraction:

1. **Name the abstraction** — Use a clear, descriptive name that reflects the shared behavior
2. **Define the interface** — Determine parameters needed to cover all usage variations
3. **Choose the location** — Follow project conventions for shared code placement
4. **List all consumers** — Identify every file that will be updated
5. **Assess risk** — Note any subtle differences between duplicated instances that need parameterization

Present the plan to the user before proceeding (unless `--dry-run` was not specified and the scope is clear).

**Plan format:**
```

## Extraction Plan


### 1. [Abstraction Name] → [target file path]
- Type: utility | component | hook
- Replaces: [N] identical blocks across [M] files
- Consumers: [list of files]
- Parameters: [any variations that need to be parameterized]
- Duplicated: [N] tokens / [N] lines (from jscpd; blank when the Grep fallback was used)
- Similarity: [N]% (from jscpd; "exact" when ast-grep-confirmed as the same shape)
- Estimated lines saved: [N]
```

The `Duplicated` and `Similarity` fields come from jscpd's report (tokens/lines per clone, and the cluster's percentage) — a quantified `--dry-run` report instead of a best-effort narrative. When the Grep fallback (1c) supplied the cluster, leave them blank or note "grep-estimated".


### Step 4: Extract shared abstractions

Execute each planned extraction:

1. **Create the shared abstraction** with proper typing and documentation
2. **Replace each instance** in consumer files with an import + usage of the new abstraction
3. **Handle variations** — parameterize differences between instances rather than creating multiple abstractions
4. **Update imports** — add the new import, remove imports that were only needed for the inline version

**Extraction order:** Start with utilities (no dependencies), then components, then hooks (may depend on utilities/components).

Mark each extraction as completed in the todo list before moving to the next.


### Step 5: Write tests

Write tests for each extracted abstraction:

| Abstraction Type | Test Approach |
|-----------------|---------------|
| Utility function | Unit tests covering all input variations, edge cases |
| UI component | Render tests, prop variations, accessibility |
| Custom hook | Hook testing with mock dependencies, state transitions |
| Type definitions | Type-level tests if applicable (tsd, expect-type) |

Place test files adjacent to the abstraction or in the project's test directory, following existing conventions.


### Step 6: Clean up dead code

After all extractions are complete:

1. **Remove unused imports** from all updated consumer files
2. **Remove dead code** — inline helper functions that are now replaced
3. **Verify no orphaned references** — search for any remaining references to removed code


### Step 7: Verify all checks pass

Run the full verification suite:

**TypeScript/JavaScript projects:**
```bash
npx tsc --noEmit          # Type checking
npm run lint              # Linting (or biome/eslint directly)
npm run test              # Full test suite
```

**Python projects:**
```bash
ty check .                # Type checking
ruff check .              # Linting
pytest                    # Test suite
```

**Rust projects:**
```bash
cargo check               # Type checking
cargo clippy              # Linting
cargo test                # Test suite
```

All three must pass. If any fail, fix the issues before reporting completion.


### Output Summary

After all phases complete, report:

```

## DRY Consolidation Summary


### Extractions
- [Abstraction Name] (type) — replaced N blocks in M files
- ...


### New Files Created
- path/to/new/file.ts — [description]
- ...


### Tests Added
- N tests across M test files


### Net Effect
- ~N lines of duplicated code consolidated
- N reusable abstractions created
- All verified: typecheck + lint + N passing tests
```


## Related Skills

- If dead code detected during consolidation → `/code:dead-code`
- If complexity is high after consolidation → `/code:complexity`
