# Ember Claude Skills — Project Contract

This repo *is* a Claude Code plugin. Every artifact here ships into other people's CLAUDE sessions, so authoring discipline matters more than in a normal codebase. Read this before adding or modifying skills, agents, or commands.

## Layout

```
ember-claude-skills/
├── CLAUDE.md                     ← this file
├── README.md                     ← user-facing docs
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── agents/                       ← subagent definitions (`.md` per agent)
├── commands/                     ← slash-command definitions (`.md` per command)
└── skills/<name>/
    ├── SKILL.md                  ← thin index + core constraints
    └── references/*.md           ← supporting files for large content
```

## Authoring rules — skills

### Frontmatter (every SKILL.md)

```yaml
---
name: <kebab-case-skill-name>             # required, must match directory
description: <one paragraph, see below>   # required
type: reference | feedback | project      # required
---
```

### Description

The description is what Claude reads to decide *whether to load this skill*. It must answer **"when should I use you,"** not "what are you."

- **Target ~30–50 tokens.** Treat 80+ as a smell. The tweet that informs this contract called 9 tokens "efficient" and 45 already "inefficient" — we're slightly looser because Ember era distinctions need keywords, but enumerating 20 addons in a description is a flag of weak categorization, not strong triggering.
- **Always include a `Use when …` clause.** That's the trigger phrase Claude matches against the user's intent.
- **Don't enumerate every concept** the skill covers. Pick the 4–6 highest-signal keywords and stop.

Good (≈45 tokens): `Authoring Glimmer components and Handlebars templates in modern Ember — args, blocks, yield, helpers, modifiers, splattributes, contextual components. Use when creating, refactoring, or reviewing any Ember component or template.`

Bad (180+ tokens): a description that lists 20 addon names trying to widen its trigger surface — split the skill or move the catalog to references/ instead.

### Body length

- **SKILL.md should stay under ~300 lines.** Treat it as navigation + core constraints, not the full reference.
- **If a skill's body would exceed 300 lines, split.** Move category-specific reference content into `skills/<name>/references/<topic>.md` and link from a catalog table in SKILL.md. See `skills/ember-ecosystem-addons/` for the canonical example.
- **Progressive disclosure is the rule.** Claude loads SKILL.md when the description matches; the supporting files are loaded only when the body links the agent to them.

### Scope

- **One skill = one job.** A skill that covers review, deploy, debug, docs, and incident triage is broken — split it.
- **Reference vs feedback vs project.** `reference` = how something works; `feedback` = practical advice / tradeoffs / when-not-to; `project` = state about a specific in-flight migration or repo. Most skills here are `reference`.
- **No side-effect skills with auto-invoke.** If a future skill *runs* something destructive (codemods, migrations applied without a worktree), set `disable-model-invocation: true` and require the user to invoke it explicitly.

## Authoring rules — agents

### Frontmatter (every agent .md)

```yaml
---
name: <kebab-case-agent-name>
description: <when to use this specific agent vs a general session>
tools: <comma-separated minimum set>
model: opus | sonnet | haiku            # only when the choice is intentional
---
```

### Tools

- **Constrain to the minimum.** A subagent should never have main-thread-wide permissions. The tweet that informs this contract is explicit: "tools / disallowedTools: 限定能用什么工具，别给和主线程一样宽的权限."
- **Don't grant `Agent`** unless the agent legitimately needs to spawn its own subagents. The migrators here don't.
- **Don't grant `Write`** to read-only audit/review agents.

### Model

- **`opus`** for migration planning, architecture decisions, anything that needs sustained reasoning across a large codebase.
- **`sonnet`** for the default. Don't set `model:` if sonnet is fine — let the field stay implicit.
- **`haiku`** for read-only exploration / scaffolding agents where speed beats depth.

### Worktree isolation

Any agent that runs `ember-cli-update`, codemods, or otherwise rewrites `package.json` / lockfiles / dozens of files in one go must include a worktree note in its body — instruct the consumer to invoke with `claude --worktree` or create one before delegating. The migrator agents are the canonical example.

## NEVER

- Stuff multi-hundred-line reference content into `SKILL.md`'s body. Split into `references/` instead.
- Write descriptions that enumerate everything the skill covers. Trim to triggers.
- Grant a subagent the same tool width as the main thread.
- Put dynamic content (timestamps, generated-on dates, current branch) into a SKILL.md or agent body — that breaks the prompt cache for every consumer.
- Recommend `find()` from `@ember/test-helpers` for assertions in skill examples — prefer `ember-cli-page-object` instead. (Repo-wide test-selector convention.)
- Use the `(mut …)` helper in template examples — write an `@action` setter and pass `this.foo` as the callback.

## ALWAYS

- Update `README.md`'s skill table when you add, rename, or remove a skill.
- Cross-link related skills under `## See also` at the bottom of each SKILL.md.
- For Polaris-era examples (5.x+), prefer `.gts` strict-mode authoring.
- For migration skills, end with a clear handoff signal: which skill or agent picks up where this one stops.

## Verification

Before committing a change:

- [ ] `find skills -name SKILL.md -exec wc -l {} +` — flag any new SKILL.md over 300 lines for splitting.
- [ ] All frontmatter has `name`, `description`, `type` (skills) or `name`, `description`, `tools` (agents).
- [ ] Description answers "when to use" with an explicit `Use when …` phrase.
- [ ] Cross-links between skills resolve (relative paths are correct).
- [ ] `README.md` table reflects the current skill / agent / command set.

## Compact instructions

When this conversation is auto-compacted, preserve in priority order:

1. **Authoring decisions** — which skills were split, which descriptions were trimmed, why a particular type was chosen.
2. **Modified files** with their key changes (especially `SKILL.md` files and frontmatter rewrites).
3. **Open TODOs** — partially-split skills, descriptions that still need trimming, missing `references/` files.
4. **Verification status** — which `wc -l` / link-check passes were green.
5. Tool outputs (full `find` / `grep` results) can be dropped; keep only the summary.
