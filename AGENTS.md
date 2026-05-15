# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Cursor, Copilot, etc.) when working with code in this repository.

## Repository Overview

A collection of skills for AI coding agents working with Redis. Skills are packaged instructions and resources that extend agent capabilities.

## Spec-Driven Workflow for Non-Trivial Changes

For non-trivial changes — new rule sets, restructuring, multi-file edits, skill enhancements, new client/SDK references — **write a numbered spec in `/spec/` first**, iterate it with reviewers until approved, **then** implement. Do not jump to code for work that touches multiple files or introduces new conventions.

### Conventions

- **Location:** `/spec/`
- **Filename:** `NNNN-kebab-topic.md` where `NNNN` is the next sequential number, zero-padded to four digits (`0001`, `0002`, …).
- **Sections expected:** problem statement, goals, non-goals, current-state inventory, proposed changes, cross-cutting decisions, acceptance criteria, open questions, out-of-scope follow-ups, iteration log.
- **Iteration log:** every meaningful spec revision is logged with a date and a one-line summary so the design history is traceable in-file.
- **Cross-references:** specs that depend on or supersede earlier specs cite them by number and link by relative path.

### When to skip the spec

For trivial changes — typo fixes, single-line edits, renames the requester already specified verbatim — go ahead and edit. The spec exists to align on direction, not to slow down small mechanical work.

### Existing specs

Numbered specs live in `/spec/`. Before starting work on Redis Search rules or RedisVL coverage, the specs in that folder are authoritative — see the next section.

## Creating a New Skill

### Directory Structure

```
skills/
  {skill-name}/               # kebab-case directory name
    SKILL.md                  # Required: skill definition with frontmatter
    AGENTS.md                 # Generated: compiled rules (for rule-based skills)
    README.md                 # Required: user documentation
    metadata.json             # Required: version and metadata
    rules/                    # For rule-based skills
      _sections.md            # Section definitions
      _template.md            # Rule template
      {prefix}-{name}.md      # Individual rules
    scripts/                  # For script-based skills (optional)
      {script-name}.sh
```

### Naming Conventions

- **Skill directory**: `kebab-case` (e.g., `redis-development`, `redis-monitoring`)
- **SKILL.md, AGENTS.md**: Always uppercase
- **Rule files**: `{prefix}-{description}.md` where prefix maps to section
- **Scripts**: `kebab-case.sh`

### SKILL.md Format

```markdown
---
name: {skill-name}
description: {One sentence describing when to use this skill. Include trigger phrases.}
license: MIT
metadata:
  author: {organization}
  version: "1.0.0"
---

# {Skill Title}

{Brief description of what the skill does.}

## When to Apply

Reference these guidelines when:
- {Use case 1}
- {Use case 2}

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | {Category} | HIGH | `{prefix}-` |

## Quick Reference

### 1. {Category} (HIGH)

- `{prefix}-{rule-name}` - {Brief description}

## How to Use

{Instructions for reading individual rules}

## Full Compiled Document

For the complete guide with all rules expanded: `AGENTS.md`
```

### Rule File Format

Each rule in `rules/` follows this structure:

```markdown
---
title: Clear, Action-Oriented Title
impact: HIGH|MEDIUM|LOW
impactDescription: Quantified benefit (e.g., "10x faster")
tags: relevant, keywords
description: Clear, Action-Oriented Title
alwaysApply: true
---

## {Title}

{1-2 sentence explanation of the problem and why it matters}

**Correct:** Description of good approach.

```python
# Good example with comments
```

**Incorrect:** Description of problematic approach.

```python
# Bad example with comments
```

Reference: [Link](URL)
```

**When to include "Incorrect" examples:**
- Use `**Incorrect:**` when the alternative causes real harm (race conditions, security issues, crashes, significant performance problems)
- For feature-introduction rules where not using the feature is valid, use `**When to use:**` / `**When NOT needed:**` sections instead
- Rules may include multiple `**Correct:**` sections for different valid approaches (see comparison rules like `json-vs-hash.md`)

### Adding a New Skill

1. Create skill directory: `skills/{skill-name}/`
2. Create `SKILL.md` with frontmatter and structure above
3. Create `metadata.json`:
   ```json
   {
     "version": "1.0.0",
     "organization": "Your Org",
     "date": "Month Year",
     "abstract": "Description for AI agents",
     "references": ["https://..."]
   }
   ```
4. Create `README.md` with user documentation
5. For rule-based skills:
   - Create `rules/_sections.md` defining categories
   - Create `rules/_template.md` for contributors
   - Add rules as `rules/{prefix}-{name}.md`
   - Add skill config to `packages/redis-development-build/src/config.ts`
   - Run `npm run build` to generate AGENTS.md

## Redis Search Skill — Specific Directives

This section applies **only to work on the `search-*` and `vector-*` rule prefixes** in `skills/redis-development/` plus the RedisVL reference. All other rule prefixes (`data-`, `ram-`, `conn-`, `json-`, `semantic-cache-`, `stream-`, `cluster-`, `security-`, `observe-`) are unaffected and continue to follow the generic skill-creation guidance above.

### Authoritative specs

Before adding, editing, or reviewing any Redis Search content, consult the specs in `/spec/`:

| Spec | Scope |
|------|-------|
| [`0001-search-syntax-coverage.md`](spec/0001-search-syntax-coverage.md) | Search & vector syntax rule set, multi-client architecture, reference-loading mechanism |
| [`0002-redis-py-client-reference.md`](spec/0002-redis-py-client-reference.md) | `references/clients/python-redis-py.md` |
| [`0003-jedis-client-reference.md`](spec/0003-jedis-client-reference.md) | `references/clients/java-jedis.md` |
| [`0004-redisvl-client-reference.md`](spec/0004-redisvl-client-reference.md) | `references/clients/python-redisvl.md` + consolidation of RedisVL code currently in `vector-*` rules |

If a Redis Search proposal is not covered by an existing spec, **write a new numbered spec first** per the workflow above; do not start coding.

### CLI-first convention

Every `search-*` and `vector-*` rule MUST lead with the canonical CLI / RESP form of the operation. Reasoning: `redis-cli` is the source of truth — every client library serializes to the same wire commands. CLI examples are universally readable, never drift, and let an agent reason about *what* Redis Search does without committing to *which* client.

Layering inside a rule:

| Layer | Location | Contents | Required? |
|-------|----------|----------|-----------|
| **L1 — Canonical CLI** | Top of every rule | RESP / `redis-cli` form of the operation | **Always** |
| **L2 — Client mirrors** | Bottom of every rule | Short `redis-py` and `Jedis` snippets (≤15 lines each) mirroring the CLI form | When meaningful; skip for CLI-only topics |
| **L3 — Per-client deep references** | `skills/redis-development/references/clients/*.md` | Comprehensive per-client coverage | Always — but only for the two clients in scope |

**No RedisVL code inline in rules.** RedisVL coverage consolidates into `references/clients/python-redisvl.md` only — see spec 0004 §6.

### Supported clients (v1)

- **`redis-py`** — Python, raw client. Reference: `references/clients/python-redis-py.md` (spec 0002).
- **Jedis** — Java. Reference: `references/clients/java-jedis.md` (spec 0003).
- **RedisVL** — higher-level Python SDK built on `redis-py`. Reference: `references/clients/python-redisvl.md` (spec 0004).

Other clients (Lettuce, node-redis, go-redis, NRedisStack, .NET) are explicitly out of scope for v1 and tracked as follow-ups in spec 0001 §10.

### Upstream "mirror, don't invent"

Every code example must trace to a Redis-maintained upstream source. Do not invent client snippets:

- **redis-py:** https://github.com/redis/redis-py/tree/master/doctests
- **Jedis:** https://github.com/redis/jedis/tree/master/src/test/java/io/redis/examples
- **RedisVL:** https://github.com/redis/redis-vl-python/tree/main/docs/user_guide

When writing a new rule:

1. Fetch the upstream file(s) listed in the relevant spec's rule-to-upstream mapping table.
2. Extract the CLI form from the inline comments or by translating the client call to its `FT.*` wire equivalent.
3. Distill the client snippet to ≤15 lines for the L2 mirror.
4. Cite the upstream filename in the rule (and the `STEP_START` label when applicable — see below).

### Shared dataset and cross-language anchors

- **Dataset:** the upstream **Bicycle dataset** (used across `redis-py` doctests and Jedis examples) is the canonical example dataset. Reuse it across rules and references. Do not invent fresh datasets unless the topic requires it (e.g., vectorizer demos may use upstream notebook data).
- **Cross-language anchors:** upstream files use `# STEP_START <label>` / `# STEP_END` (Python) and `// STEP_START <label>` / `// STEP_END` (Java) to delimit teachable steps. Preserve the **same step labels** across the two clients so an agent can pair-verify behavior cross-language.

### Product naming

- The official product is **Redis Search**, not "Redis Query Engine" / "RQE."
- Filenames use the `search-*` prefix (not `rqe-*`).
- Titles, headings, and prose use "Redis Search"; "search" is used as a verb.

### `references/` folder pattern

Redis Search introduces a new `references/` folder under `skills/redis-development/` for progressive-disclosure docs that are loaded on demand (not always-loaded like rules):

```
skills/redis-development/
├── rules/                                  (always-loaded behavior — CLI-first)
├── references/
│   ├── README.md                           (router: task → reference)
│   ├── search-syntax-primitives.md         (query DSL vocabulary)
│   └── clients/
│       ├── python-redis-py.md
│       ├── java-jedis.md
│       └── python-redisvl.md
```

### Conditional reference loading

References are **loaded conditionally** by the consuming agent, not bulk-included in `AGENTS.md`. Three coordinated mechanisms (spec 0001 §7.8):

1. **Frontmatter scope** on each reference file declares `scope`, `triggerWhen`, and `appliesTo` (e.g., `appliesTo.client: redis-py`).
2. **Rule directive blocks** at the bottom of every rule with client mirrors instruct the agent which reference to load and explicitly warn against loading mismatched ones:

   ```markdown
   **Client mirrors — read exactly one:**
   - For raw redis-py targets, read `references/clients/python-redis-py.md`.
   - For Jedis (Java) targets, read `references/clients/java-jedis.md`.
   - For RedisVL targets, read `references/clients/python-redisvl.md`.
   - Do not read more than one client reference.
   ```

3. **Router** (`references/README.md`) maps task signal → reference path for agents that start from the `references/` folder.

The build inlines only the router table at the top of `AGENTS.md`; reference *bodies* stay out. A build-time validator requires the directive block in every rule that includes client mirrors.

### Version gates

- **FT.HYBRID:** requires Redis ≥ 8.4.0 (Redis Open Source). Rules using FT.HYBRID must include a version gate; fall back to the pre-filter + KNN pattern via FT.SEARCH for older Redis. Verified syntax in spec 0001 §5.0a.
- **DIALECT:** the spec assumes DIALECT 2 (Redis 8+ default). Older Redis defaults vary by client version; rules touching DIALECT call out client-version differences.

### How to contribute a new Redis Search rule

1. Confirm the rule is covered by an existing spec (or write a new one — see Spec-Driven Workflow).
2. Author the rule as CLI-first (L1), then add short `redis-py` and Jedis mirrors (L2). Cite upstream files.
3. Add the conditional-loading directive block at the bottom.
4. Add the rule to the relevant section under `skills/redis-development/SKILL.md` Quick Reference.
5. Run `npm run validate` and `npm run build` from the repo root. The validator enforces the directive block, frontmatter shape, and CLI-first structure.

## Build System

The build tooling in `packages/redis-development-build/` compiles individual rule files into AGENTS.md:

```bash
npm install
npm run validate  # Check rule structure
npm run build     # Generate AGENTS.md
```

To add a new skill to the build system, update `src/config.ts`:

```typescript
export const SKILLS: Record<string, SkillConfig> = {
  'your-new-skill': {
    name: 'your-new-skill',
    title: 'Your New Skill',
    description: 'Description',
    skillDir: join(SKILLS_DIR, 'your-new-skill'),
    rulesDir: join(SKILLS_DIR, 'your-new-skill/rules'),
    metadataFile: join(SKILLS_DIR, 'your-new-skill/metadata.json'),
    outputFile: join(SKILLS_DIR, 'your-new-skill/AGENTS.md'),
    sectionMap: {
      prefix1: 1,
      prefix2: 2,
    },
  },
}
```

### Best Practices for Context Efficiency

- **Keep SKILL.md under 500 lines** — put detailed rules in separate files
- **Write specific descriptions** — helps agents know when to activate the skill
- **Use progressive disclosure** — SKILL.md summarizes, AGENTS.md has full details
- **Quantify impact** — "10x faster" helps agents prioritize rules

### End-User Installation

**Claude Code:**
```bash
cp -r skills/{skill-name} ~/.claude/skills/
```

**Claude.ai:**
Add SKILL.md to project knowledge or paste contents into conversation.
