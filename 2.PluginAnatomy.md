# 02 — Plugin anatomy: file layout and manifests

> What Claude Code looks for when you install a plugin, what each manifest contains, and how `${CLAUDE_PLUGIN_ROOT}` lets you reference assets without hardcoding paths.

---

## The bird's-eye layout

Here is the entire `a11y-workflow-marketplace/` directory, the part that Claude Code actually reads:

```
a11y-workflow-marketplace/
├── .claude-plugin/
│   └── marketplace.json          ← MARKETPLACE manifest (catalog of plugins)
├── plugins/
│   └── a11y/                     ← one plugin lives here
│       ├── .claude-plugin/
│       │   └── plugin.json       ← PLUGIN manifest (this single plugin's metadata)
│       ├── .mcp.json             ← MCP servers bundled with the plugin
│       ├── agents/
│       │   ├── a11y-investigator.md
│       │   ├── a11y-developer.md
│       │   └── a11y-shipper.md
│       ├── commands/
│       │   ├── investigate.md
│       │   ├── developer.md
│       │   └── ship.md
│       ├── skills/
│       │   ├── a11y-browser-verify/SKILL.md  (+ references/, scripts/)
│       │   ├── expert/SKILL.md  (+ rules/)
│       │   ├── lint/SKILL.md
│       │   ├── localize/SKILL.md
│       │   ├── test/SKILL.md  (+ references/, scripts/)
│       │   └── ux/SKILL.md  (+ rules/, scripts/)
│       ├── knowledge/
│       │   ├── README.md
│       │   ├── investigator-field-template.md
│       │   ├── developer-field-template.md
│       │   ├── principles.md
│       │   ├── products.md
│       │   ├── themes.md
│       │   └── attachment-analysis.md
│       └── scripts/
│           ├── README.md
│           └── axe.min.js        ← vendored axe-core (4.11.4)
└── sync-plugin.sh                ← optional: source → cache sync helper
```

Claude Code recognises four runtime artifact types: `agents/`, `commands/`, `skills/`, and `.mcp.json`. Everything else (`knowledge/`, `scripts/`, `references/` inside skills) is convention — folders the plugin's own code points at. Claude Code doesn't traverse them automatically.

---

## The two manifests

### `marketplace.json` — the catalog

`a11y-workflow-marketplace/.claude-plugin/marketplace.json:1–17`:

```json
{
    "name": "a11y-workflow-marketplace",
    "owner": {
        "name": "Thomson Reuters",
        "email": "cobalt@thomsonreuters.com"
    },
    "plugins": [
        {
            "name": "a11y",
            "source": "./plugins/a11y",
            "description": "WCAG accessibility auditing and fixing plugin for TR",
            "version": "1.2.0",
            "author": { "name": "Rajath Rao" },
            "category": "productivity"
        }
    ]
}
```

This is what `/plugin marketplace add <path>` consumes. A marketplace can list multiple plugins; this one ships exactly one. Each entry has:

| Field | What it does |
| --- | --- |
| `name` | The handle users type — `/plugin install a11y@a11y-workflow-marketplace`. |
| `source` | Relative path to the plugin's source directory. The path is resolved relative to the `marketplace.json` file. |
| `version` | SemVer. Used by Claude Code's cache directory layout (`~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/`). |
| `description` / `author` / `category` | Metadata shown in install UIs and footers. |

The marketplace pattern is what lets you publish a "catalog" repo (one Git repo with many plugins) instead of one repo per plugin.

### `plugin.json` — the plugin's own metadata

`a11y-workflow-marketplace/plugins/a11y/.claude-plugin/plugin.json:1–8`:

```json
{
    "name": "a11y",
    "description": "WCAG accessibility auditing and fixing plugin for Thomson Reuters Frontend Repos.",
    "version": "1.2.0",
    "author": {
        "name": "Thomson Reuters Frontend Repos"
    }
}
```

Tiny on purpose. It restates the same `name` / `version` that the marketplace already declares. Claude Code uses it as the authoritative source for the *installed* plugin's identity (especially when a plugin is installed without a marketplace).

The two `version` strings should match. The `sync-plugin.sh` helper script (covered below) reads `plugin.json:version` to figure out which cache directory to sync to.

---

## How Claude Code finds the runtime artifacts

Once the plugin is installed (`/plugin install a11y@a11y-workflow-marketplace`), Claude Code scans the plugin's root for the four primitives:

| Folder / file | Resolves into | Discovery rule |
| --- | --- | --- |
| `agents/*.md` | Subagents available to the `Agent` tool. The `subagent_type` becomes `<plugin-name>:<agent-name>`, e.g., `a11y:a11y-developer`. | One agent per file. Filename (without `.md`) defaults to the subagent name; YAML frontmatter `name:` overrides. |
| `commands/*.md` | Slash commands. `/a11y:investigate`, `/a11y:developer`, `/a11y:ship`. | One command per file. Plugin name becomes the namespace. |
| `skills/<name>/SKILL.md` | Skills. Some are user-invokable as slash commands (`/a11y:a11y-browser-verify`); others are reasoning helpers loaded by name. | One skill per directory. `SKILL.md` is the entry point. |
| `.mcp.json` | MCP servers bundled with this plugin. | Single file at plugin root. Its `mcpServers` are merged with user / project-level MCP servers at runtime. |

`knowledge/`, `scripts/`, and `references/` *inside* skills aren't discovered automatically. They exist because some artifact (an agent's system prompt, a skill's body) explicitly says "Read this path." The path is built using `${CLAUDE_PLUGIN_ROOT}` — see below.

---

## `${CLAUDE_PLUGIN_ROOT}` — the asset path lever

Every Claude Code plugin lives in *two* directories at runtime:

1. **Source** — where you cloned / authored it (`a11y-workflow/a11y-workflow-marketplace/plugins/a11y/`).
2. **Cache** — where Claude Code copies it on install (`~/.claude/plugins/cache/a11y-workflow-marketplace/a11y/<version>/`).

Agents and skills run against the *cache* copy, not your source. So when an agent says "Read this knowledge file," it can't write a relative path — relative to *what*? It also can't hardcode an absolute path — every user has a different home directory.

The variable `${CLAUDE_PLUGIN_ROOT}` is Claude Code's solution. At runtime, it expands to the absolute path of the cache directory of *this plugin*. The agent says:

```
Read ${CLAUDE_PLUGIN_ROOT}/knowledge/investigator-field-template.md
```

…and the harness expands it to (e.g.) `C:/Users/C304925/.claude/plugins/cache/a11y-workflow-marketplace/a11y/1.2.0/knowledge/investigator-field-template.md`.

Every internal asset reference in this plugin uses `${CLAUDE_PLUGIN_ROOT}`:

- Agents reading knowledge files (`agents/a11y-investigator.md:111`, `agents/a11y-developer.md:82–83`).
- The Playwright MCP server's `--init-script` argument (`.mcp.json:20`).
- Skill bodies that point at sibling reference files (`skills/a11y-browser-verify/SKILL.md:274`).

**Why this matters in practice.** If you ever see a path that starts with anything other than `${CLAUDE_PLUGIN_ROOT}` or a project-relative `<RUN_DIR>` reference inside a plugin asset, that's a bug — it'll break the moment someone runs the plugin from a different directory or a different cache version.

---

## The install / activate flow

From a clean machine to a working plugin (`docs/setup-guide.md:178–204`):

```text
1. /plugin marketplace add <absolute-path>/a11y-workflow/a11y-workflow-marketplace
2. /plugin install a11y@a11y-workflow-marketplace
3. /reload-plugins
```

`/plugin marketplace add` reads `marketplace.json` and registers the catalog. `/plugin install` copies the plugin source into the cache, parses the manifests, and registers the artifacts. `/reload-plugins` re-scans on demand without a Claude Code restart.

If everything worked, the Claude Code footer shows something like:

```
... 3 agents · 3 commands · 6 skills · 2 plugin MCP servers ...
```

— which is the plugin's contribution: 3 agents, 3 slash commands, 6 skills (`a11y-browser-verify` + `expert` + `lint` + `localize` + `test` + `ux`), and 2 MCP servers from the bundled `.mcp.json` (Saffron and Playwright).

If you see lower numbers, your `/reload-plugins` didn't pick up the source. The next section explains why.

---

## Source vs cache, and `sync-plugin.sh`

Editing a plugin and not seeing your changes is the most common confusion. Cause: you edited the source, but Claude Code is reading the cache.

There are two ways to handle this:

**Option A — install from source path directly.** Some Claude Code versions support pointing the install at the source dir; reloads then re-read source on every reload. Verify with `/plugin info`.

**Option B — sync source → cache.** Copy edits from source to cache and reload. The plugin ships a helper for this:

`a11y-workflow-marketplace/sync-plugin.sh:1–63` (excerpted):

```bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SRC="$SCRIPT_DIR/plugins/a11y"
MARKETPLACE_NAME="a11y-workflow-marketplace"
CACHE="$HOME/.claude/plugins/cache/$MARKETPLACE_NAME/a11y"

CURRENT_VERSION=$(grep -o '"version"[[:space:]]*:[[:space:]]*"[^"]*"' \
    "$SRC/.claude-plugin/plugin.json" | head -1 | sed 's/.*"\([^"]*\)"$/\1/')

for VERSION in $VERSIONS; do
    DEST="$CACHE/$VERSION"
    for dir in skills agents commands knowledge scripts .claude-plugin; do
        if [ -d "$SRC/$dir" ]; then
            mkdir -p "$DEST/$dir"
            cp -r "$SRC/$dir/." "$DEST/$dir/"
        fi
    done
    for f in "$SRC"/.mcp.json "$SRC"/*.md; do
        [ -f "$f" ] && cp "$f" "$DEST/"
    done
done
```

Three things this script does that are worth copying:

1. **Auto-locates itself** via `BASH_SOURCE` so it doesn't matter what your working directory is when you run it.
2. **Reads the plugin's own version** from `plugin.json` to know which cache subdirectory to target.
3. **Iterates *every* cached version**, not just the current one — useful when you have multiple plugin versions installed and you want all of them updated to your local edits.

You only need this pattern if you're authoring or rapidly iterating on a plugin. End users install via `/plugin install` and never touch `sync-plugin.sh`.

---

## Frontmatter — the convention every artifact uses

Agents, commands, and skills are all markdown files with YAML frontmatter at the top. The frontmatter declares metadata; the body is the system prompt / instructions.

Example — minimal agent (`agents/a11y-shipper.md:1–20`):

```yaml
---
name: a11y-shipper
description: |
    Ships an accessibility fix as a GitHub draft pull request. ...
allowed-tools:
    - Read
    - Write
    - Bash
model: opus
effort: high
initialPrompt: |
    Step 1: Read <RUN_DIR>/run-state.json. Validate preconditions ...
---
```

Example — minimal slash command (`commands/ship.md:1–4`):

```yaml
---
description: Ship an accessibility fix produced by the developer. ...
argument-hint: '<bug-id>'
---
```

Example — minimal skill (`skills/a11y-browser-verify/SKILL.md:1–5`):

```yaml
---
name: a11y-browser-verify
description: Run a live WCAG 2.1 AA accessibility audit on a web page using Playwright MCP + axe-core. ...
argument-hint: "[<url>]"
---
```

Each primitive has its own frontmatter fields. The most important:

| Field | Used by | Effect |
| --- | --- | --- |
| `name` | agents, skills | Override the file-name-derived identifier. |
| `description` | all three | Shown in `/help`, `/agents`, the Claude Code footer. **Also used by Claude Code to decide when to auto-trigger a skill** — write it in keyword-rich terms (see the `a11y-browser-verify` description). |
| `allowed-tools` | agents | Restricts what the agent can call. The smaller the better. |
| `argument-hint` | commands, skills | Help text for `/<command>`. |
| `model` | agents, skills | `opus` / `sonnet` / `haiku`. Default: inherit. |
| `effort` | agents | Roughly maps to thinking depth. `high` for the heavy reasoners. |
| `initialPrompt` | agents | Pre-canned message injected as the first user turn when the agent starts. Used here to enforce mandatory-first-action reads. |

Doc 04 (Agents) covers `allowed-tools`, `effort`, and `initialPrompt` in depth. Doc 05 (Commands) covers `argument-hint`. Doc 06 (Skills) covers the user-invokable vs reasoning-helper distinction.

---

## What's NOT in the plugin

Worth naming, because every other plugin tutorial implies the opposite:

- **No build step.** No `tsconfig`, no Webpack, no `dist/`. The plugin is markdown + JSON + one vendored JS file (axe-core). `npm install` is not a thing here.
- **No `package.json` at the plugin level.** The plugin doesn't have npm deps of its own. The `.mcp.json` *invokes* `npx -y @playwright/mcp@latest`, which fetches Playwright on demand at MCP startup, but the plugin doesn't ship `node_modules`.
- **No tests.** The plugin's "tests" are the field templates (output schemas the agents must match) and the self-check steps in agent prompts. There's no automated CI on the plugin's markdown.
- **No persistent server.** Everything runs on demand — agents are spawned by `Agent` tool calls, MCP servers spawn when first used. There's no daemon to install.

That minimalism is the point. The plugin is a *folder of markdown* — easy to read, easy to fork, easy to diff in code review.

---

**Next:** [03 — State machine](03-state-machine.md). The single shared file (`run-state.json`), the phase lock, and the ownership model that lets three agents collaborate without overwriting each other.
