# /learn — building Claude Code plugins, by reading this one

> A pedagogical deep-dive into the `a11y` plugin. Use it as the worked example to learn how Claude Code plugins are designed, then scaffold your own.

---

## What this is

10 short topical docs that walk through every piece of the `a11y` plugin and explain the design decisions behind it. Concept first ("what is a phase lock?"), then concrete ("here's how `agents/a11y-developer.md:79–86` enforces it"). By the end you'll have built a minimal sibling plugin from scratch.

Audience: someone who has used Claude Code as a user but hasn't authored a non-trivial plugin. No prior plugin-authoring experience required.

---

## Reading order

Read in order if you're new. Each doc points to the next:

1. **[01 — Overview](01-overview.md)** — The problem the plugin solves and the shape of the solution. The 30-second pitch.
2. **[02 — Plugin anatomy](02-plugin-anatomy.md)** — Manifest files, runtime artifact types, `${CLAUDE_PLUGIN_ROOT}`, install flow.
3. **[03 — State machine](03-state-machine.md)** — `run-state.json`, the phase lock, ownership model, atomic single-write.
4. **[04 — Agents](04-agents.md)** — Subagent design: frontmatter, tool grants, mandatory-first-action reads, structured refusals, error budgets.
5. **[05 — Slash commands](05-slash-commands.md)** — Orchestrator pattern, `AskUserQuestion` gating, git lifecycle, `--chat` / `--revise` / `--bg` modes.
6. **[06 — Skills](06-skills.md)** — User-invokable vs reasoning-helper roles. The Playwright + axe-core injection pattern. Script-augmented skills.
7. **[07 — Knowledge base](07-knowledge-base.md)** — Four genres of `knowledge/` files. Output contracts, universal reasoning, project data, prompt guides.
8. **[08 — MCP integration](08-mcp-integration.md)** — Bundled vs user-level servers, tool-name namespacing, the `--init-script` lever.
9. **[09 — Design patterns](09-design-patterns.md)** — 18 generalizable patterns: name → problem → solution → anchor → reuse guidance.
10. **[10 — Build your own](10-build-your-own.md)** — Hands-on scaffold of `mini-a11y`: manifest, agent, command, knowledge, skill, MCP. ~60 minutes.

---

## Jump to a topic

If you're looking for a specific question:

| Question | Doc |
| --- | --- |
| Why are there three agents instead of one? | [01](01-overview.md), [03](03-state-machine.md) |
| What files does Claude Code actually read when it loads a plugin? | [02](02-plugin-anatomy.md) |
| What does `${CLAUDE_PLUGIN_ROOT}` resolve to? | [02](02-plugin-anatomy.md) |
| Why one shared state file instead of agent-to-agent payloads? | [03](03-state-machine.md) |
| Why does the developer agent write `run-state.json` only once? | [03](03-state-machine.md), pattern 2 in [09](09-design-patterns.md) |
| How do I set up an agent's `allowed-tools`? | [04](04-agents.md) |
| Why must the agent's first tool call be a Read of a markdown file? | [04](04-agents.md), pattern 4 in [09](09-design-patterns.md) |
| How does the diff-review gate work? Approve / Adjust / Discard? | [05](05-slash-commands.md) |
| Why is git in the slash command, not the agent? | [05](05-slash-commands.md) |
| How does axe-core get into every browser page? | [06](06-skills.md), [08](08-mcp-integration.md) |
| When do I write a skill vs a slash command vs an agent? | [06](06-skills.md) |
| Where does project-specific data go? | [07](07-knowledge-base.md) |
| What's `FORBIDDEN FIELDS` and why? | [07](07-knowledge-base.md), pattern 7 in [09](09-design-patterns.md) |
| Should I bundle this MCP server in the plugin or leave it at user level? | [08](08-mcp-integration.md), pattern 17 in [09](09-design-patterns.md) |
| How does tool namespacing work for plugin-bundled MCP servers? | [08](08-mcp-integration.md) |
| What patterns can I lift from this plugin into mine? | [09](09-design-patterns.md) |
| Show me how to scaffold a plugin from scratch. | [10](10-build-your-own.md) |

---

## Prerequisites

- You've used Claude Code as a user. You know what slash commands and `/<plugin>:<command>` look like.
- You're comfortable with markdown frontmatter (YAML at the top of a markdown file).
- Basic familiarity with JSON. You don't need to be a JSON expert.
- A Claude Code installation you can run plugins on (for the doc 10 walkthrough).

What you don't need: Anthropic SDK / API knowledge, prior plugin authorship experience, deep WCAG / accessibility expertise. The plugin is the worked example; the patterns are general.

---

## What's NOT in /learn

These docs complement, not replace, the existing repo docs:

- **`docs/setup-guide.md`** — installation procedures and troubleshooting. We point at it for setup specifics; we don't duplicate it.
- **`docs/state.md`** — current architecture and command reference. Same — referenced, not duplicated.
- **`README.md` (top-level)** — project status and roadmap. We quote it; we don't restate it.

These docs also do **not** cover:

- WCAG content beyond the conceptual minimum needed to understand what the agents are doing.
- Saffron design system internals (the developer agent calls Saffron MCP; how Saffron *itself* works is out of scope).
- Anthropic API / SDK usage. The plugin runs entirely inside Claude Code.

If you want WCAG depth, read `plugins/a11y/knowledge/principles.md` and the rest of the knowledge base directly. If you want Saffron, see `Ask Saffron MCP Server` documentation (TR-internal).

---

## How long will this take

| Reading | Time |
| --- | --- |
| Skim the 10 docs | ~30 minutes |
| Read all 10 thoughtfully | ~2 hours |
| Read + do the doc-10 walkthrough | ~3 hours total (60 min for the walkthrough) |
| Use this as ongoing reference while authoring your own plugin | indefinite — return for specific topics |

If you have an hour: read 01 (overview) and 09 (patterns), skim 03 (state machine) and 04 (agents). That's the conceptual core.

If you have 3 hours: read everything in order, do the walkthrough.

---

## A note on style

These docs cite real lines in the codebase using `path:line-line` notation, e.g., `agents/a11y-developer.md:79–86`. When you see those, open the actual file at those lines — the citation isn't decoration, it's the source. The docs paraphrase only when they introduce a concept; for actual rules and contracts they quote.

Quoted blocks come from real plugin files, verbatim. If you spot a discrepancy between a doc and the source, **trust the source.** Patches welcome.

---

Start with [01 — Overview](01-overview.md).
