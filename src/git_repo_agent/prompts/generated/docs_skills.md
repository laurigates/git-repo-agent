## configure-readme

## The Slim Shape

The README is the only template. It is read on the repository page, on package
registries and often inside release archives, so it stays short and points into
`docs/` for everything else.

| README.md keeps | `docs/` gets (only pages that have content) |
|-----------------|---------------------------------------------|
| Header: logo or banner, title, badges | `usage.md` — command line, configs, outputs, recipes |
| What the project does, in two or three sentences | `configuration.md` — options, environment variables, config files |
| Quick start: install, one example command, an outputs table where the tool writes files | `architecture.md` (or `pipeline.md`) — stages, components, technology |
| Documentation index: a table linking each `docs/` page | `development.md` — building, checks, project layout, contributing, releases |
| Licence | |

Sections such as Features, Tech Stack, Usage reference, Configuration,
Architecture, Project Structure, Development, Contributing and Releases belong
in `docs/`. A README that carries them is non-compliant, and the fix moves them.


## Execution

Execute this README configuration workflow:


### Step 1: Detect project metadata

Read name, description, version, licence, repository and keywords from the
detected package file (`package.json`, `pyproject.toml`, `Cargo.toml`, or the
module path in `go.mod`). With no package file, fall back to the directory name,
the first line of the existing README, the git remote URL and the LICENSE file.
Package file formats are in .


### Step 2: Inventory the README and docs/

1. Read README.md and classify each `##` section with the section-to-page map in
   : **keep** (header, description, quick start,
   documentation index, licence) or **move** (and to which `docs/` page).
2. Check the kept parts: description is two or three sentences; quick start has
   an install command and one example command; the documentation index links
   every `docs/` page; badges and a licence are present.
3. Read the existing `docs/*.md` pages so moved content merges into them instead
   of duplicating.
4. Find files that point at README sections:
   `rg -n 'README(\.md)?(#|\s*§)' --glob '!node_modules'` plus prose such as
   "see the README". Each hit that names a section being moved needs repointing.


### Step 3: Report compliance

Run the link check and print the compliance report (format in
): PASS / PARTIAL / MISSING for each kept part, and
`MOVE → docs/<page>.md` for each detail section still in the README.

```bash
bash "${CLAUDE_SKILL_DIR}/scripts/check-readme-links.sh" --project-dir "$(pwd)"
```

If `--check-only` is set, stop here.


### Step 4: Move the detail sections into docs/

Unless `--fix` is set, confirm the planned moves with report to orchestrator first.

1. Create only the `docs/` pages that receive content; a project with no
   configuration gets no `configuration.md`.
2. Move each section's text as written, demoting headings one level under the
   page's `#` title. Merge into an existing page rather than creating a second
   one on the same topic.
3. Generate the project layout for `development.md` with
   `tree -L 2 -I 'node_modules|target|__pycache__|.git|dist|build' --dirsfirst`.
4. Fix relative links inside moved text: a link that was `src/x.py` from the
   root is `../src/x.py` from `docs/`.


### Step 5: Write the slim README

Write README.md from the slim template in : header
with logo (unless `--no-logo` or no asset exists) and badges, two or three
sentences, quick start, documentation index table, licence. Take the install and
example commands from the detected stack:

| Package Manager | Install | Run | Test | Build |
|----------------|---------|-----|------|-------|
| npm/bun (package.json) | Read `scripts` from package.json | `npm run dev` | `npm test` | `npm run build` |
| uv/poetry (pyproject.toml) | `uv sync` / `poetry install` | `uv run python -m pkg` | `uv run pytest` | - |
| cargo (Cargo.toml) | `cargo install --path .` | `cargo run` | `cargo test` | `cargo build --release` |
| go (go.mod) | `go install ./...` | `go run .` | `go test ./...` | `go build` |

Test and build commands go in `docs/development.md`, not the README.


### Step 6: Repoint references and verify links

1. Update every hit from Step 2.4 to the section's new page, for example
   `README.md#configuration` becomes `docs/configuration.md`, and prose that
   cites a README section by name cites the new page instead.
2. Re-run `check-readme-links.sh` and fix each `broken_link` / `broken_anchor`
   row until it reports `STATUS=OK`.


### Step 7: Update standards tracking

Update `.project-standards.yaml`, and remove a `readme_style` key left by an
earlier version of this skill (the slim shape is the only template):

```yaml
components:
  readme: "2026.11"
  readme_has_logo: true|false
  readme_badges: ["license", "ci", "version"]
  readme_docs_pages: ["usage", "development"]
```


### Step 8: Report results

Print the files written, the sections moved and where, the references repointed,
and the final link-check status. Results format in .


## Output

Provide:
1. Compliance report against the slim shape
2. The slim README.md and each `docs/` page written or updated
3. References repointed, and the link-check result
4. Suggestions (logo, badges, missing quick-start example)


# configure-readme Reference

Slim README and `docs/` page templates, the section-to-page map, badge
reference, and report formats. (Absorbed the former `readme-standards`
reference skill.)


## Why slim

A long README (features, tech stack, getting started, usage, structure,
development, contributing) grows past 200 lines on a project with real users.
It is hard to scan, and the same file is rendered on package registries and
often shipped inside release archives. The slim README answers three questions
(what is it, how do I try it, where is the rest) and links into `docs/` for
everything else. One user-facing CLI went from 215 README lines to 65 with this
split.


## Section-to-Page Map

Classify each `##` section of an existing README by its heading and content.
Create a `docs/` page only when at least one section maps to it.

| README section (common headings) | Action | Destination |
|----------------------------------|--------|-------------|
| Logo, title, tagline, badges | keep | README header |
| Description, About, Overview | keep, trim to 2–3 sentences | README |
| Installation, Getting Started, Quick Start | keep install + one example; move the rest | README Quick Start, remainder to `docs/usage.md` |
| Documentation, Docs | keep, rebuild as the index table | README |
| License, Licence | keep | README |
| Features, Highlights | move | `docs/usage.md` (intro) or `docs/architecture.md` when it describes internals |
| Usage, CLI, Commands, Examples, Recipes, Outputs | move | `docs/usage.md` |
| Configuration, Options, Environment Variables, Settings | move | `docs/configuration.md` |
| Architecture, Pipeline, How It Works, Tech Stack, Design | move | `docs/architecture.md` (or `docs/pipeline.md` for a staged tool) |
| Project Structure, Layout | move | `docs/development.md` |
| Development, Building, Testing, Linting, Checks | move | `docs/development.md` |
| Contributing, Releases, Changelog | move | `docs/development.md` (link CONTRIBUTING.md / CHANGELOG.md when they exist) |
| API, API Reference | move | `docs/api.md`, or the generated reference from `/configure:docs` |
| Demo, Screenshots | keep one screenshot or link in the header; move galleries | `docs/usage.md` |
| Status, Roadmap, FAQ, Troubleshooting | move | `docs/usage.md` (FAQ, troubleshooting) or `docs/development.md` (status, roadmap) |

An unknown heading goes to the page whose topic it shares; ask with
report to orchestrator when two pages fit equally.


## Slim README Template

````markdown
<div align="center">

<img src="assets/logo.png" alt="PROJECT_NAME logo" width="128" height="128">


# PROJECT_NAME

[![License](https://img.shields.io/github/license/OWNER/REPO)](LICENSE)
[![CI](https://img.shields.io/github/actions/workflow/status/OWNER/REPO/ci.yml?branch=main&label=CI)](https://github.com/OWNER/REPO/actions)
[![Release](https://img.shields.io/github/v/release/OWNER/REPO)](https://github.com/OWNER/REPO/releases)

</div>

PROJECT_NAME does WHAT for WHOM. One sentence on how it works or what makes it
different. Optionally one sentence on scope or status.


## Quick Start

```bash
INSTALL_COMMAND
EXAMPLE_COMMAND
```

| Output | Contents |
|--------|----------|
| `OUTPUT_FILE_1` | What the file holds |
| `OUTPUT_FILE_2` | What the file holds |


## Documentation

| Page | Contents |
|------|----------|
| [Usage](docs/usage.md) | Command line, configs, outputs, recipes |
| [Configuration](docs/configuration.md) | Options and environment variables |
| [Architecture](docs/architecture.md) | Stages, components, technology |
| [Development](docs/development.md) | Building, checks, layout, contributing, releases |


## License

[LICENSE_TYPE](LICENSE)
````

Template rules:

- Drop the outputs table when the tool writes no files (a library, a service).
- List only the `docs/` pages that exist in the documentation index.
- A multi-step install (for example a Homebrew tap: `brew tap`, then
  `brew install`) stays in the Quick Start block; alternative install methods
  go to `docs/usage.md`.
- Library: the example is a three-to-five-line import-and-call snippet in the
  project's language instead of a shell command.
- Web application: the example is the run command plus the local URL; a live
  demo link goes in the header.


## docs/ Page Templates

Each page opens with a `#` title and a one-line statement of what it covers.
Moved sections keep their text and drop one heading level.


### docs/usage.md

````markdown

# Usage

How to run PROJECT_NAME: the command line, configs, outputs and common recipes.
Start with the [Quick Start](../README.md#quick-start) if you have not
installed it yet.


## Command line

```bash
PROJECT_NAME <command> [options]
```

| Command | Purpose |
|---------|---------|
| `COMMAND` | What it does |


## Outputs


## Recipes
````


### docs/configuration.md

````markdown

# Configuration

Every option PROJECT_NAME reads, where it reads it from, and the defaults.

| Option | Environment variable | Default | Purpose |
|--------|----------------------|---------|---------|
````


### docs/architecture.md

````markdown

# Architecture

How PROJECT_NAME is built: its stages or components and the technology behind
them.


## Stages


## Technology

| Category | Technology |
|----------|------------|
````


### docs/development.md

````markdown

# Development

Building PROJECT_NAME from source, running the checks, the project layout,
and how changes are contributed and released.


## Building

```bash
INSTALL_DEPS_COMMAND
BUILD_COMMAND
```


## Checks

```bash
TEST_COMMAND
LINT_COMMAND
```


## Layout

```
PROJECT_NAME/
├── src/      # Source code
├── tests/    # Tests
└── docs/     # Documentation
```


## Contributing


## Releases
````


## Link Check

`scripts/check-readme-links.sh --project-dir <dir>` resolves every relative
link in the project's Markdown files (README.md, `docs/`, and any other `*.md`,
such as skills that cite README sections). It skips external URLs, code spans,
fenced code, and `node_modules`/`vendor`/`target`/`dist`/`build`/`.venv`.

| Key | Meaning |
|-----|---------|
| `FILES_SCANNED` | Markdown files read |
| `LINKS_CHECKED` | Relative links resolved |
| `BROKEN_LINKS` | Links whose file or directory does not exist |
| `BROKEN_ANCHORS` | Links whose `#anchor` matches no heading (GitHub slug rules) or `<a id>` in the target |
| `STATUS` | `OK` (exit 0) or `ERROR` (exit 1); `REASON=` names the first finding |

Each finding is an `ISSUES:` row with `TYPE=broken_link|broken_anchor|readme_missing`,
`FILE=`, `LINE=` and `TARGET=`. The usual fix after a split is to point the link
at the new page: `README.md#configuration` becomes `docs/configuration.md`.


## Package File Format Examples


### package.json (JavaScript/TypeScript)

```javascript
{
  "name": "project-name",
  "description": "Project description",
  "version": "1.0.0",
  "license": "MIT",
  "repository": { "url": "github:owner/repo" },
  "keywords": ["keyword1", "keyword2"]
}
```


### pyproject.toml (Python)

```toml
[project]
name = "project-name"
description = "Project description"
version = "1.0.0"
license = "MIT"
keywords = ["keyword1", "keyword2"]

[project.urls]
Repository = "https://github.com/owner/repo"
```


### Cargo.toml (Rust)

```toml
[package]
name = "project-name"
description = "Project description"
version = "1.0.0"
license = "MIT"
repository = "https://github.com/owner/repo"
keywords = ["keyword1", "keyword2"]
```


### go.mod (Go)

```
module github.com/owner/repo
```


## Compliance Report Format

```
README.md Compliance Report (slim shape)
========================================
Project: [name]
README lines: [N]

Kept in README:
  Header (logo/banner)   [PASS | MISSING | N/A (no assets)]
  Title                  [PASS | MISSING]
  Badges                 [PASS | PARTIAL | MISSING]
  Description            [PASS | TOO LONG | MISSING]   (2–3 sentences)
  Quick start            [PASS | PARTIAL | MISSING]    (install + one example)
  Documentation index    [PASS | INCOMPLETE | MISSING] (links every docs/ page)
  License                [PASS | MISSING]

Detail sections still in README:
  Features               MOVE → docs/usage.md
  Tech Stack             MOVE → docs/architecture.md
  Development            MOVE → docs/development.md

docs/ pages:
  docs/usage.md          [EXISTS | WILL CREATE]
  docs/development.md    [EXISTS | WILL CREATE]

Links (check-readme-links.sh):
  STATUS=[OK | ERROR]  BROKEN_LINKS=[N]  BROKEN_ANCHORS=[N]

References to README sections outside README.md: [N]

Overall: [X issues found]
```

A quick start that has an install command but no example command is
`PARTIAL`, not `MISSING`.


## Results Report Format

```
README.md Configuration Results
===============================
Project: [name]
README: [old] → [new] lines

Moved:
  Features, Usage        → docs/usage.md (created)
  Tech Stack             → docs/architecture.md (created)
  Development, Contributing, Project Structure → docs/development.md (merged)

README now holds: header, badges, description, quick start, documentation index, license

References repointed:
  skills/status/SKILL.md:12  README.md#status → docs/development.md#status

Link check: STATUS=OK (LINKS_CHECKED=[N])

Next Steps:
  - Review the moved text for sentences that assumed the README context
  - Add a logo or banner to assets/ (optional)
  - Run /configure:github-pages to publish docs/ as a site (optional)
```


## Badge URL Patterns


### Standard Shields.io Badges

| Badge Type | URL Pattern |
|------------|-------------|
| License | `https://img.shields.io/github/license/OWNER/REPO` |
| Stars | `https://img.shields.io/github/stars/OWNER/REPO` |
| Forks | `https://img.shields.io/github/forks/OWNER/REPO` |
| Issues | `https://img.shields.io/github/issues/OWNER/REPO` |
| CI Status | `https://img.shields.io/github/actions/workflow/status/OWNER/REPO/WORKFLOW.yml` |
| Version | `https://img.shields.io/github/v/release/OWNER/REPO` |
| npm | `https://img.shields.io/npm/v/PACKAGE` |
| PyPI | `https://img.shields.io/pypi/v/PACKAGE` |
| Crates.io | `https://img.shields.io/crates/v/PACKAGE` |


### Runtime/Tech Badges

| Technology | Badge URL |
|------------|-----------|
| Bun | `https://img.shields.io/badge/runtime-Bun-f9f1e1` |
| Node.js | `https://img.shields.io/badge/node-%3E%3D20-brightgreen` |
| TypeScript | `https://img.shields.io/badge/TypeScript-5.x-blue` |
| Python | `https://img.shields.io/badge/python-3.12-blue` |
| Rust | `https://img.shields.io/badge/rust-1.75+-orange` |
| Go | `https://img.shields.io/badge/go-1.22-00ADD8` |

Keep the badge row to three to five badges; a version, CI and licence badge
cover most projects.


## Logo Guidelines

- **Format**: PNG (with transparency) or SVG
- **Size**: 128x128px to 512x512px; a banner can be wider
- **Location**: `assets/logo.png` or `assets/icon.svg`
- **Centering**: `<div align="center"><img src="assets/logo.png" alt="Project Name" width="128"></div>`


### Placeholder Logo

When no logo exists, use a centered title, optionally with a project-type emoji:

```markdown
<div align="center">


# 🚀 PROJECT_NAME

</div>
```

| Emoji | Project type |
|-------|--------------|
| 🚀 | General/deployment tools |
| 🛠️ | Developer tools |
| 📊 | Data/analytics |
| 🔒 | Security |
| 🌐 | Web applications |
| 📱 | Mobile apps |
| 🤖 | AI/ML projects |
| 📦 | Package/library |

Tools for creating logos:
- [Shields.io](https://shields.io) for custom badges
- [Simple Icons](https://simpleicons.org) for technology icons
- AI image generators for custom logos


## Section Writing Standards


### Description — say what it does, for whom

**Good:**
```markdown
widget-cli turns scanner output into searchable photo albums. It discovers
eSCL scanners on the local network and crops each photo out of a multi-photo
scan.
```

**Compare — implementation-focused (less effective):**
```markdown
A Rust CLI using mDNS and an edge-detection algorithm.
```


### Features in docs/usage.md — write benefits, not implementation

```markdown
- **Automatic Scanner Detection** - Discovers eSCL-compatible scanners via mDNS without manual configuration
- **Smart Photo Separation** - Detects and crops multiple photos from a single scan
```


### Layout in docs/development.md — keep it scannable

- 2-3 levels deep, only meaningful directories, brief comments for
  non-obvious folders


## Cookiecutter Integration

For creating entire new projects from templates, consider
[cookiecutter](https://cookiecutter.readthedocs.io/) (`uv tool install
cookiecutter`, then `cookiecutter <template-url>`). Cookiecutter is ideal for
organization-wide templates and whole-project scaffolding; `/configure:readme`
is better for updating existing projects and compliance-checking existing
READMEs.

---

## blueprint-claude-md

Generate or update the project's CLAUDE.md file based on blueprint artifacts, PRDs, and project structure.

Branch-specific detail lives in `references/`, split by the path that needs it — an invocation opens only the file for the branch it takes.
## CLAUDE.md vs Auto Memory

Claude Code has two complementary systems for project context. CLAUDE.md should contain **team-shared instructions** — not patterns Claude learns on its own.

| Belongs in CLAUDE.md | Belongs in Auto Memory (managed by Claude) |
|----------------------|---------------------------------------------|
| Team coding standards | Debugging insights and workarounds |
| Build/test/lint commands | Personal workflow preferences |
| Architecture decisions | Project-specific patterns learned over time |
| Required conventions | File relationships and navigation shortcuts |
| CI/CD workflows | Common mistakes and how to fix them |

Auto memory lives at `~/.claude/projects/<project>/memory/` and is managed automatically. Do not duplicate auto memory concerns into CLAUDE.md.


## Memory Hierarchy (precedence low → high)

1. **User-level rules**: `~/.claude/rules/` — personal rules across all projects
2. **CLAUDE.md (project)**: Team-shared project instructions (checked into git)
3. **CLAUDE.local.md**: Personal project-specific preferences (gitignored)
4. **.claude/rules/**: Modular, path-specific rules
5. **Managed policy**: Organization-wide instructions (enterprise, system paths)
6. **Auto memory**: Claude's own notes (`~/.claude/projects/<project>/memory/`)


## @import Syntax

CLAUDE.md files support importing other markdown files to stay lean:

```markdown

# Project: MyApp

@docs/architecture.md
@docs/conventions.md
@.claude/rules/testing.md
```

- Paths are relative to the file containing the import
- Recursive imports supported (max depth 5)
- Imports are not evaluated inside code spans or code blocks
- First-time external imports trigger an approval dialog

Use `@import` to reference existing documentation rather than duplicating content into CLAUDE.md.


## CLAUDE.md Best Practices

- Keep it concise (< 500 lines ideally)
- Focus on team-shared instructions (standards, commands, architecture)
- Use `@import` to reference existing docs instead of duplicating content
- Use `CLAUDE.local.md` for personal preferences (auto-gitignored)
- Reference `.claude/rules/` for detailed, path-specific rules
- Let auto memory handle "Current Focus", "Key Files", debugging tips
- Update when PRDs change significantly


## Execution

Execute this CLAUDE.md workflow:


### Step 1: Check current state

- Look for existing `CLAUDE.md` in project root
- Look for existing `CLAUDE.local.md` (personal preferences, gitignored)
- Read `docs/blueprint/manifest.json` for configuration
- Check for `~/.claude/rules/` (user-level rules)
- Determine `claude_md_mode` (single, modular, or both)


### Step 2: Determine action

Use report to orchestrator:

```
{If CLAUDE.md exists:}
question: "CLAUDE.md already exists. What would you like to do?"
options:
  - "Update with latest project info" → merge updates
  - "Regenerate completely" → overwrite (backup first)
  - "Add missing sections only" → append new content
  - "Add @imports for existing docs" → replace inline content with imports
  - "Convert to modular rules" → split into .claude/rules/
  - "Create CLAUDE.local.md" → personal preferences (gitignored)
  - "View current structure" → analyze and display

{If CLAUDE.md doesn't exist:}
question: "No CLAUDE.md found. How would you like to create it?"
options:
  - "Generate from project analysis" → auto-generate
  - "Generate from PRDs" → use blueprint PRDs
  - "Generate with @imports (lean)" → auto-generate using imports for existing docs
  - "Start with template" → use starter template
  - "Use modular rules instead" → skip CLAUDE.md, use rules/
```


### Step 3: Gather project context

- **Project structure**: Detect language, framework, build tools
- **PRDs**: Read `docs/prds/*.md` for requirements
- **Work overview**: Current phase and progress
- **Existing rules**: Content from `.claude/rules/` if present
- **Git history**: Recent patterns and conventions
- **Dependencies**: Package managers, key libraries


### Step 4: Generate CLAUDE.md sections

Build the sections from [`references/template-sections.md`](references/template-sections.md): the standard skeleton, tailored with the per-project-type additions. Leave out what auto memory handles (see "CLAUDE.md vs Auto Memory" above).


### Step 5: Apply the modular rules mode

If modular rules mode is `both` or `modular`, follow the matching section of [`references/modular-rules-split.md`](references/modular-rules-split.md). Mode `single` skips this step.


### Step 6: Apply a selected option

- "Create CLAUDE.local.md" → [`references/template-sections.md`](references/template-sections.md) § CLAUDE.local.md template
- "Add @imports for existing docs" → [`references/smart-update.md`](references/smart-update.md) § Add @imports


### Step 7: Smart update (existing CLAUDE.md)

When CLAUDE.md already exists, follow [`references/smart-update.md`](references/smart-update.md) § Smart update.


### Step 8: Sync with modular rules

When `.claude/rules/` has rules, follow [`references/modular-rules-split.md`](references/modular-rules-split.md) § Sync with modular rules.


### Step 9: Update manifest and task registry

Record the run in `docs/blueprint/manifest.json` per [`references/manifest-and-registry.md`](references/manifest-and-registry.md).


### Step 10: Report

```
✅ CLAUDE.md updated!

{Created | Updated}: CLAUDE.md
{If created:} CLAUDE.local.md (personal preferences, gitignored)

Sections:
- Overview ✅
- Tech Stack ✅
- Development Workflow ✅
- Architecture ✅
- Conventions ✅

@imports used: {count, if any}
- @docs/prds/architecture.md
- @.claude/rules/testing.md

Sources used:
- PRDs: {list}
- Rules: {list}
- Project detection: {what was detected}

{If modular mode:}
Note: Detailed rules are in .claude/rules/
CLAUDE.md serves as overview and quick reference.

Note: "Current Focus" and "Key Files" are managed by Claude's
auto memory — no need to maintain these in CLAUDE.md.

Run `/blueprint:status` to see full configuration.
```


### Step 11: Prompt for next action

Use report to orchestrator:

```
question: "CLAUDE.md updated. What would you like to do next?"
options:
  - label: "Check blueprint status (Recommended)"
    description: "Run /blueprint:status to verify configuration"
  - label: "Manage modular rules"
    description: "Add or edit rules in .claude/rules/"
  - label: "Continue development"
    description: "Run /project:continue to work on next task"
  - label: "I'm done for now"
    description: "Exit - CLAUDE.md is saved"
```

**Based on selection:**
- "Check blueprint status" → Run `/blueprint:status`
- "Manage modular rules" → Run `/blueprint:rules`
- "Continue development" → Run `/project:continue`
- "I'm done" → Exit

---

## blueprint-docs-list

List blueprint documents programmatically from the filesystem. Extracts metadata from YAML frontmatter and markdown headers.
## Execution


### If arg is `adrs`

Run `/blueprint:adr-list` — it handles ADR-specific extraction with both header-section and frontmatter support.


### If arg is `prds`

```bash
printf "| PRD | Title | Status | Date |\n|-----|-------|--------|------|\n" && \
for f in docs/prds/*.md; do
  [ -f "$f" ] || continue
  fname=$(basename "$f")
  [ "$fname" = "README.md" ] && continue
  doc_title=$(head -50 "$f" | grep -m1 "^title:" | sed 's/^title:[[:space:]]*//' || true)
  doc_status=$(head -50 "$f" | grep -m1 "^status:" | sed 's/^status:[[:space:]]*//' || true)
  doc_date=$(head -50 "$f" | grep -m1 "^date:\|^created:" | sed 's/^[^:]*:[[:space:]]*//' || true)
  # Fallback: extract title from H1
  if [ -z "$doc_title" ]; then
    doc_title=$(head -20 "$f" | grep -m1 "^# " | sed 's/^# //')
  fi
  printf "| [%s](%s) | %s | %s | %s |\n" \
    "${fname%.md}" "$f" "${doc_title:-(untitled)}" "${doc_status:--}" "${doc_date:--}"
done
```


### If arg is `prps`

```bash
printf "| PRP | Title | Status | Confidence |\n|-----|-------|--------|------------|\n" && \
for f in docs/prps/*.md; do
  [ -f "$f" ] || continue
  fname=$(basename "$f")
  [ "$fname" = "README.md" ] && continue
  doc_title=$(head -50 "$f" | grep -m1 "^title:" | sed 's/^title:[[:space:]]*//' || true)
  doc_status=$(head -50 "$f" | grep -m1 "^status:" | sed 's/^status:[[:space:]]*//' || true)
  doc_confidence=$(head -50 "$f" | grep -m1 "^confidence:" | sed 's/^confidence:[[:space:]]*//' || true)
  if [ -z "$doc_title" ]; then
    doc_title=$(head -20 "$f" | grep -m1 "^# " | sed 's/^# //')
  fi
  printf "| [%s](%s) | %s | %s | %s |\n" \
    "${fname%.md}" "$f" "${doc_title:-(untitled)}" "${doc_status:--}" "${doc_confidence:--}"
done
```


### If arg is `all`

Show counts and status breakdown for each document type:

```bash
echo "## Blueprint Documents Summary"
echo ""
for doc_type in adrs prds prps; do
  doc_count=$(ls docs/$doc_type/*.md 2>/dev/null | grep -cv 'README.md' || echo 0)
  doc_label=$(printf '%s' "$doc_type" | tr '[:lower:]' '[:upper:]')
  echo "### ${doc_label}: $doc_count documents"
  if [ "$doc_count" -gt 0 ]; then
    for s in Accepted Proposed Deprecated Superseded draft ready approved; do
      sc=$(grep -ril "^status:.*$s\|^$s$" docs/$doc_type/*.md 2>/dev/null | wc -l | tr -d ' ')
      [ "$sc" -gt 0 ] && echo "- $s: $sc"
    done
  fi
  echo ""
done
```


## Post-Actions

After listing, suggest:
- For empty directories: "Run `/blueprint:derive-plans` to generate documents"
- For stale documents: "Review documents with status 'draft' or 'Proposed'"

---

## blueprint-curate-docs

# /blueprint:curate-docs

Curate library or project documentation into `.claude/rules/` entries optimized for AI agents — concise, actionable, gotcha-aware context that is auto-loaded into sessions and referenceable from PRPs.

> **Note:** earlier blueprint versions wrote these entries to `docs/blueprint/ai_docs/`. That location is deprecated — `.claude/rules/` is the canonical home for curated AI context, since rules are loaded by Claude Code natively and shared with every contributor.

**Usage**: `/blueprint:curate-docs [library-name]` or `/blueprint:curate-docs project:[pattern-name]`


## Execution

Execute complete documentation curation workflow:


### Step 1: Determine target and check existing rules

1. Parse argument to determine if library or project pattern
2. Resolve `$RULES_DIR` (manifest `structure.generated_rules_path`, default `.claude/rules/`)
3. Check if a rule for this library/pattern already exists in `$RULES_DIR`
4. If exists → Ask: Update in place or skip?
5. Check project dependencies for library version


### Step 2: Research and gather documentation

For **libraries**:
- Find official documentation URL
- Search for specific sections relevant to project use cases
- Find known issues and gotchas (WebSearch: "{library} common issues", "{library} gotchas")
- Extract key sections with WebFetch

For **project patterns**:
- Search codebase for pattern implementations: `grep -r "{pattern}" src/`
- Identify where and how it's used
- Document conventions and variations
- Extract real code examples from project


### Step 3: Extract key information

1. **Use cases**: How/why this library/pattern is used in project
2. **Common operations**: Most frequent uses
3. **Patterns we use**: Project-specific implementations (with file references)
4. **Configuration**: How it's configured in this project
5. **Gotchas**: Version-specific behaviors, common mistakes, performance pitfalls, security considerations

Sources for gotchas: GitHub issues, Stack Overflow, team experience, official docs warnings.


### Step 4: Create the rule entry

Generate the file at `$RULES_DIR/lib-[library-name].md` or `$RULES_DIR/[pattern-name].md` (see [REFERENCE.md](REFERENCE.md#template)).

**Never overwrite a hand-authored rule.** If a file with the target name exists and was not produced by this skill, pick a distinct name (e.g. `lib-[library-name]-gotchas.md`) or ask.

Include all sections from template: Quick Reference, Patterns We Use, Configuration, Gotchas, Testing, Examples.

Keep under 200 lines total.


### Step 5: Add code examples

Include copy-paste-ready code snippets from:
- Project codebase (reference actual files and line numbers)
- Official documentation examples
- Stack Overflow solutions
- Personal implementation experience


### Step 6: Update task registry

Update the task registry entry in `docs/blueprint/manifest.json`:

```bash
jq --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --argjson processed "${ITEMS_PROCESSED:-0}" \
  --argjson created "${ITEMS_CREATED:-0}" \
  '.task_registry["curate-docs"].last_completed_at = $now |
   .task_registry["curate-docs"].last_result = "success" |
   .task_registry["curate-docs"].stats.runs_total = ((.task_registry["curate-docs"].stats.runs_total // 0) + 1) |
   .task_registry["curate-docs"].stats.items_processed = $processed |
   .task_registry["curate-docs"].stats.items_created = $created' \
  docs/blueprint/manifest.json > tmp.json && mv tmp.json docs/blueprint/manifest.json
```


### Step 7: Validate and save

1. Verify entry is < 200 lines
2. Verify all code examples are accurate
3. Verify gotchas include solutions
4. Save file
5. Report completion, including the rule path so PRPs can reference it


# blueprint-curate-docs REFERENCE


## Template

Curated entries live in the project's rules directory (`structure.generated_rules_path`, default `.claude/rules/`) — `lib-[library-name].md` for libraries, `[pattern-name].md` for project patterns.

```markdown

# [Library/Pattern Name]

**Version:** X.Y.Z
**Last Updated:** YYYY-MM-DD
**Use Case:** [Why we use this in this project]


### [Common Operation 1]
\`\`\`language

# Code snippet that can be directly copied
\`\`\`


## Patterns We Use


### [Pattern Name]
[When to use this pattern]

\`\`\`language

# Full example as used in this project

# Reference: src/path/file.ts:15-25
\`\`\`


## Configuration

\`\`\`
VAR_NAME=value
\`\`\`


## Gotchas


### Gotcha 1: [Title]
**Issue:** [What can go wrong]
**Solution:**
\`\`\`language

# Correct approach
\`\`\`


## Testing


### How to Mock
\`\`\`language

# Mocking pattern for tests
\`\`\`


## References
- [Official docs](https://...)
- [Related code](src/path/file.ts)
```


## Section Guidelines

| Section | Guidelines | Lines |
|---------|---|---|
| Quick Reference | Most common operations, copy-paste ready | 20-30 |
| Patterns We Use | Project-specific implementations with line refs | 30-50 |
| Configuration | Environment variables and project config | 10-20 |
| Gotchas | Known issues with solutions | 40-60 |
| Testing | Mocking and test patterns | 20-30 |
| References | Links to docs and related code | 5-10 |


## Line Count Target
< 200 lines total. Concise and actionable, not comprehensive documentation.


## Code Example Format

Include file/line references:
```typescript
// Reference: src/auth/handlers.ts:42-55
function login(req, res) {
  // Implementation
}
```


## Gotcha Format

Always include solution:
```

### Gotcha: Connection pool exhaustion
**Issue:** Pool defaults to 10, causes timeouts under load
**Solution:** Set pool size to CPU count × 4
```

---

## code-docs-quality

Analyze and validate documentation quality for a codebase, ensuring PRDs, ADRs, PRPs, CLAUDE.md, and .claude/rules/ are up to standards and current.
## Your Task

Perform a comprehensive documentation quality analysis using the following methodology:


## Phase 1: Plan the analysis

Track the analysis against this checklist (via `TodoWrite` when the session
has the task tools — see `.claude/rules/agentic-permissions.md` § Task-tool
availability — otherwise carry it as a checklist in your response):

```
- Analyze CLAUDE.md structure and quality
- Check .claude/rules/ directory and standards
- Validate ADRs (Architecture Decision Records)
- Validate PRDs (Product Requirements Documents)
- Validate PRPs (Product Requirement Prompts)
- Check documentation freshness and git history
- Generate quality report with recommendations
```


## Phase 2: CLAUDE.md Analysis


### 2.1 Check Existence and Structure
- Verify `CLAUDE.md` exists at project root
- Check for required YAML frontmatter:
  ```yaml
  ---
  created: YYYY-MM-DD
  modified: YYYY-MM-DD
  reviewed: YYYY-MM-DD
  ---
  ```


### 2.2 Content Quality Checks
- **Completeness**: Does it provide clear project context?
  - Project structure overview
  - Development conventions
  - Key rules and guidelines
  - Plugin/tool references if applicable

- **Clarity**: Is it well-organized and readable?
  - Clear sections with headers
  - Proper markdown formatting
  - Tables for structured data
  - Examples where helpful

- **Accuracy**: Check for outdated information
  - Compare modified date with recent git commits
  - Look for references to deprecated patterns
  - Validate file paths and references exist


### 2.3 Best Practices
Check against CLAUDE.md standards:
- Should guide AI assistants on how to work with the codebase
- Include project structure explanation
- Reference key rules and conventions
- Point to additional documentation (ADRs, rules)
- Use tables for quick reference
- Keep focused and concise


## Phase 3: .claude/rules/ Analysis


### 3.1 Directory Structure
- Verify `.claude/rules/` directory exists
- List all rule files present
- Check for recommended core rules:
  - Architecture/design patterns
  - Coding standards
  - Development workflows
  - Testing requirements


### 3.2 Individual Rule Validation
For each rule file in `.claude/rules/`:

**Required Frontmatter**:
```yaml
---
created: YYYY-MM-DD
modified: YYYY-MM-DD
reviewed: YYYY-MM-DD
---
```

**Content Standards**:
- Clear title and purpose
- Well-defined scope
- Specific, actionable guidance
- Examples where appropriate
- Related rules cross-referenced


### 3.3 Rules Organization
- Are rules properly categorized?
- Are there too many/too few rules?
- Is there duplication or conflict between rules?
- Are rule names descriptive and consistent?


## Phase 4: ADR Validation


### 4.1 Check ADR Directory
- Verify `docs/adrs/` or `docs/adr/` exists
- Check for ADR index/README
- List all ADR files (should be numbered: 0001-title.md)


### 4.2 ADR Structure Validation
For each ADR, verify naming and the required MADR sections against
[references/doc-standards.md](references/doc-standards.md#42-adr-structure-validation).


### 4.3 ADR Quality Checks
- **Status accuracy**: Are deprecated ADRs marked?
- **Completeness**: Do ADRs have all required sections?
- **Relevance**: Are ADRs still applicable to current codebase?
- **Index maintenance**: Is the ADR index up to date?


### 4.4 Coverage Analysis
- Do ADRs cover major architectural decisions?
- Are recent significant changes documented?
- Are there obvious undocumented decisions?


## Phase 5: PRD Validation


### 5.1 Check PRD Directory
- Verify `docs/prds/` or `.claude/blueprints/prds/` exists
- List all PRD files


### 5.2 PRD Structure Validation
For each PRD, verify frontmatter and required sections against
[references/doc-standards.md](references/doc-standards.md#52-prd-structure-validation).


### 5.3 PRD Quality Checks
- **Clarity**: Are requirements specific and measurable?
- **Completeness**: Are all key sections present?
- **Status**: Is the status field accurate?
- **Traceability**: Can requirements be traced to implementations?
- **User focus**: Are user needs clearly articulated?


## Phase 6: PRP Validation


### 6.1 Check PRP Directory
- Verify `docs/prps/` exists (Blueprint methodology)
- List all PRP files


### 6.2 PRP Structure Validation
For each PRP, verify the required sections against
[references/doc-standards.md](references/doc-standards.md#62-prp-structure-validation).


### 6.3 PRP Quality Checks
- **Specificity**: Are file paths and code references explicit?
- **Testability**: Are success criteria measurable?
- **Completeness**: Does the confidence score match content quality?
- **Actionability**: Can this PRP be executed immediately?
- **Context**: Is there enough curated context for implementation?


## Phase 7: Freshness Analysis


### 7.1 Check Last Modified Dates
For all documentation:
- Compare `modified` frontmatter dates with git history
- Identify stale documents (>6 months without review)
- Flag documents that should be reviewed based on related code changes

Use git to check activity:
```bash

# Check recent commits affecting docs
git log --since="6 months ago" --oneline -- docs/ .claude/ CLAUDE.md 2>/dev/null || echo "Not a git repo or no history"


# Check when documentation was last touched
git log -1 --format="%ai %s" -- CLAUDE.md 2>/dev/null || echo "No git history"
```


### 7.2 Cross-Reference with Code Changes
- Have there been major code changes without doc updates?
- Are ADRs current with actual architecture?
- Do PRDs reflect implemented features accurately?


## Phase 8: Generate Quality Report

Build the report in four parts — 8.1 documentation inventory table, 8.2 quality
score (0-10 per category, with rating guide), 8.3 issues categorized as Critical /
Warnings / Suggestions, 8.4 actionable recommendations — using the templates in
[references/report-templates.md](references/report-templates.md).


## Phase 9: Present Results


### 9.1 Executive Summary
Show a clear, concise summary using the executive-summary template in
[references/report-templates.md](references/report-templates.md#91-executive-summary).


### 9.2 Full Report
Present the complete analysis with:
- Documentation inventory
- Quality scores by category
- Detailed findings
- Actionable recommendations
- Maintenance checklist


### 9.3 Guidance for Improvement

Help the user understand next steps:
- If Blueprint not initialized → suggest `/blueprint:init`
- If ADRs missing → suggest `/blueprint:derive-plans`
- If PRDs missing → suggest `/blueprint:derive-plans`
- If documentation tooling not configured → suggest `/configure:docs`
- If documentation outdated → provide update checklist
- If standards not followed → show examples and templates


## Best Practices


### For the Analysis
- Be thorough but concise
- Focus on actionable issues
- Provide specific file/line references
- Include examples of good vs bad patterns
- Suggest concrete fixes, not just problems


### For Documentation Standards
The general standards to recommend (frontmatter dates, templates, maintenance
cadence) are in [references/doc-standards.md](references/doc-standards.md#for-documentation-standards).


## Error Handling

- If not a git repository → skip git-based freshness checks
- If directories don't exist → note in report as missing
- If files malformed → flag specific parsing errors
- If unable to read file → note permission/access issues
- If Blueprint not initialized → suggest initialization before creating docs


## Output Format

Use clear markdown formatting:
- Tables for structured data
- Checkboxes for action items
- Code blocks for examples
- Emoji indicators (✅ ❌ ⚠️) for quick scanning
- Collapsible sections for detailed analysis (if supported)

**Remember**: The goal is to help users maintain high-quality, current documentation that serves both human developers and AI assistants effectively.
