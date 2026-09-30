---
layout: post
title: "AGENTS.md, Rules, Skills: A Repo Harness for a Coding Agent"
tags: [agents, skills, architecture, ai]
author: Nicolas Mugnier
categories: ai
description: "The model is rented. The harness is yours. AGENTS.md is the always-on map, rules are one constraint plus one reason, skills are the workflows you got tired of retyping."
locale: en_US
---

A coding agent is a model plus a harness. Everyone rents the model from the same vendors. The harness is yours.

Part of the harness is the tool (Codex, Claude Code, Cursor, a local Hermes). The other part lives in the repo: what the agent reads *before* it writes, and what tells it that it was wrong *after*. I follow Alfonso Graziano's split in [How to set up your repository for AI coding agents](https://ainativesoftware.engineering/harness){:target="_blank"}. This post is only the first half, the files. Gates another day.

The shape I keep:

```
AGENTS.md
.agents/
  skills/
    add-endpoint/
      SKILL.md
  rules/
    money.md
    api.md
    testing.md
```

Three jobs. Not three folders for the same text.

---

## Three layers, three context bills

**AGENTS.md** is the always-on contract. It is read on (almost) every session, so it costs tokens even for "fix this typo". [agents.md](https://agents.md/){:target="_blank"}: free-form markdown, read by a couple of dozen agents, nearest file wins in a monorepo. Graziano, from GitHub on about 2,500 files: commands, tests, a map of the repo, style, Git, boundaries. Four tests per line: can I *catch* the agent that breaks it; every *don't* has a *do*; it came from a real failure (or removing it would cause one); the file stays under 150-200 lines. Past that, a link, not a paste.

**A rule** is a constraint, not a workflow. It applies when it applies. Off-session, it should cost nothing.

**A skill** is the procedure you used to retype. Graziano: "Always validate inputs with Zod" is a rule. The full process for a new endpoint is a skill. The harness puts only the name and description in context (~50-100 tokens). The body arrives if the task matches.

Mixing the three means paying the instruction curse twice.

---

## The curse, and why I do not stack

Harada et al., via Graziano: if a model follows one rule at 95%, the probability of following *all* of them is that rate to the power of the count. Five rules: ~77%. Ten: ~60%. Twenty: ~36%.

Every always-on sentence *lowers* the chance that the rest is followed. A `.agents/rules/` that you `@`-import from AGENTS.md is a second AGENTS.md. You might as well paste 400 lines into the first one.

So: few invariants in AGENTS.md. Everything else loads on demand, or never.

---

## How to write a rule

One constraint, one reason. Not an essay. Not "keep it maintainable".

Template (EARS):

- the system shall X
- when Y, the system shall X
- if W, the system shall X

Then **because** &lt;operational reason&gt;. The reason does two jobs: the agent can judge a case you did not list, and you *delete* the rule the day the reason dies.

Bad:

> Use Decimal, not float.

Good:

> When persisting or computing money, the system shall use Decimal, never float, because IEEE rounding leaks into invoice totals and a type checker will not catch it.

Layers, not a pile: base → language → framework → project. A PHP service and a React app share the base, not the rest. If several repos share the base, version it and pin it.

Wording has to be testable. "Catch specific exceptions, not generic ones" passes. "Write clean code" does not.

---

## `.agents/rules/`: one concern per file

One file = one *concern*, kebab-case. Nest only if the subdomain is real. Not a 400-line `frontend.md`.

```
.agents/rules/
  money.md
  testing.md
  git.md
  api.md
```

Optional frontmatter, shape of the [agent-rules-spec RFC](https://github.com/rameshsunkara/agent-rules-spec){:target="_blank"} (close to Cursor `.mdc`):

```markdown
---
description: Money. Decimal, never float. Load on billing/pricing files.
trigger: auto
paths:
  - "src/billing/**"
  - "src/**/*pricing*"
---

When persisting or computing money, the system shall use Decimal, never float, because IEEE rounding leaks into invoice totals and a type checker will not catch it.
```

- `trigger: always`: 3 to 7 silent invariants. Secrets, money, "never touch a migration without asking".
- `trigger: auto` + `paths`: the lever. Off those files, the rule costs nothing.
- `trigger: manual`: rare. If it is a workflow, it is a skill.

Keep out of rules: build commands (AGENTS.md), a 12-step checklist (skill), anything the linter or CI already enforces. That is the *feedback* half of the harness.

---

## AGENTS.md points. It does not include

Red Hat is blunt: orientation table, not a dump. Graziano: past 150-200 lines, link out.

In AGENTS.md, for me:

1. **Boundaries in three tiers** (always / ask first / never). Graziano puts them here, not in rules: this is action routing, not style. "Ask first" for things that are legitimate but expensive to undo (migration, lockfile).
2. 4 to 6 **exact commands**. `phpunit --filter Foo` or `pnpm test --run --reporter=dot`, not "run the tests".
3. A *when to read what* table:

```markdown
## Rules (load on demand, do not ingest the folder)

- money / invoices → `.agents/rules/money.md`
- HTTP API → `.agents/rules/api.md`
- tests → `.agents/rules/testing.md`
```

The agent that does not scan `.agents/rules/` still has the pointer. The one that scans can ignore the table. Both stay coherent.

Skills: **do not** list every skill. The catalog and the `description` field on `SKILL.md` exist for that. One line at most:

```markdown
Project skills live in `.agents/skills/`. Load by description. Do not paste them here.
```

---

## Skills: the workflow you retype

The convention that is settling: `.agents/skills/<name>/SKILL.md` ([Red Hat](https://developers.redhat.com/articles/2026/07/27/standardize-project-context-agentsmd-and-agent-skills){:target="_blank"}, Agent Skills). One folder, one `SKILL.md`, optionally `scripts/`, `references/`, `assets/`. Progressive disclosure: metadata always, body on demand.

The `description` is the trigger. Vague, and the skill never loads, or it loads every time. One sentence, the *when*, not a novel.

Body example (not a rule):

```markdown
---
name: add-endpoint
description: Add an HTTP endpoint with validation, test, and default-off flag.
---

1. Copy the spec template.
2. Scaffold from the generator.
3. Validate the body at the service layer, not only in the route.
4. Handle empty results explicitly. Do not return a zero.
5. Add an integration test against seeded data.
6. Register behind a flag that defaults to off.
```

Install an external skill: read the markdown *and* every script. Skills in the repo go through PR, like code.

---

## The duplication trap

The same sentence in AGENTS.md *and* a rule is not useful redundancy. After a refactor, the agent picks one. You will not know which.

One sentence of discipline:

> AGENTS.md is the map. Rules are one constraint and one reason, scoped so they stay out of unrelated sessions. Skills are the workflows I got tired of retyping.

---

## Honesty about `.agents/rules/`

AGENTS.md and `.agents/skills/` are the portable pieces today.

`.agents/rules/` is the right *shape*. It is not yet a standard that everything reads. Cursor reads `.cursor/rules/*.mdc` (globs). Claude Code reads `.claude/rules/*.md`. The RFC proposes `.agents/rules/` as a single source; it is not deployed everywhere.

I put the source of truth in `.agents/`. If a tool does not scan that path, a thin adapter (copy, symlink, generator) beats three texts that drift. Until then, nested `AGENTS.md` (nearest wins) is the *already* standardized mechanism for scope.

I do not write a `CLAUDE.md` that repeats AGENTS.md. An `@AGENTS.md` plus Claude-only bits, if I have any. Cursor also reading `CLAUDE.md` as always-on is a documented leak: anything that must not contaminate Cursor does not go in that file.

---

## AGENTS.md skeleton

Short on purpose. Fill the `<>` per repo.

```markdown
# AGENTS.md

## Stack

- Language / runtime: <e.g. PHP 8.3, Node 22>
- Tests: <exact command>
- Lint / types: <exact command>
- One local check: <e.g. make check>

## Layout

<10 lines max. What ls does not tell you.>

## Boundaries

- Always: <silent invariants, 3-5 bullets>
- Ask first: migrations, lockfile, secrets, infra
- Never: <protected paths, prod, credentials>

## Rules (load on demand, do not ingest the folder)

- money / invoices → `.agents/rules/money.md`
- HTTP API → `.agents/rules/api.md`
- tests → `.agents/rules/testing.md`

Project skills live in `.agents/skills/`. Load by description. Do not paste them here.

## Git

- <commit / PR title format, if it is actually enforced>
```

---

## What this post does not do

Graziano's second half: a `make check` (lint, types, tests, build), deterministic gates, LLM review, human review. Without that, you brief an agent that opens PRs nobody has time to read.

I have not put this tree in a production repo yet. The shape is fixed. The next step is a real repository, not a second markdown of principles.

---

## References

- [Alfonso Graziano: How to set up your repository for AI coding agents](https://ainativesoftware.engineering/harness){:target="_blank"}
- [agents.md](https://agents.md/){:target="_blank"}
- [Red Hat: Standardize project context with AGENTS.md and Agent Skills](https://developers.redhat.com/articles/2026/07/27/standardize-project-context-agentsmd-and-agent-skills){:target="_blank"}
- [RFC: agent-rules-spec](https://github.com/rameshsunkara/agent-rules-spec){:target="_blank"}
- [agents.md issue 179](https://github.com/agentsmd/agents.md/issues/179){:target="_blank"}
