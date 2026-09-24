# JacPi

An AST-edits-first coding agent for the [Jac](https://www.jaseci.org) language,
built on [pi](https://github.com/badlogic/pi-coding-agent).

Regular coding agents edit files by matching text and patching line offsets.
JacPi removes that failure mode: the **`edit` tool is disabled** and every code
change goes through `jac_ast_edit` — symbol-targeted tree-sitter operations
that locate code by name, splice at exact AST positions, and verify the result
by re-parsing before it ever touches disk.

## How it edits

```
symbols          jac_ast_edit              jac_check_syntax      jac_format_jac
(map targets) →  (surgical AST ops,      →  (semantic verify)   →  (canonical style)
                  re-parse gate on write)
```

1. **Discover** — `{path, action:'symbols'}` lists every symbol (kind,
   qualified name, line range). The model picks targets by name, never by line.
2. **Edit** — 27 typed operations across four tiers (MICRO / BODY / STRUCT /
   FILE): `rename`, `set_type`, `add_parameter`, `set_body`, `replace_in_body`
   anchor pairs, `add_method`, `add_import`/`organize_imports`, declaration
   creators (`add_archetype`, `add_impl`, `add_test`, …). Multi-op batches are
   atomic; a final re-parse gate rejects the batch if syntax regresses.
3. **Verify** — the jac MCP server (`jac mcp`) provides
   `jac_check_syntax`, `jac_validate_jac`, `jac_lint_jac`.
4. **Format** — `jac_format_jac` canonicalizes style (the engine is
   deliberately formatting-agnostic).

Miss a name? The error *is* the search: a full available-symbols list plus a
Damerau-Levenshtein "Did you mean?" (`Card.labl` → `Card.label`).

## Components

| Piece | Repo / path | Role |
|---|---|---|
| Editing engine | [chess10kp/pi-jac-ast-edit](https://github.com/chess10kp/pi-jac-ast-edit) | tree-sitter binding + `jac_ast_edit` pi tool (27 ops) |
| This repo | config + launcher | the agent itself: isolated agent dir, tool allowlist, jac MCP, system prompt |
| `jacpy` | `~/.zshrc` function | launches the agent **from your current project**: `PI_CODING_AGENT_DIR=.pi-home` + explicit `-e` extension load — no `cd`, sessions/AGENTS.md/git stay on your repo |

Tool surface (verified): `read`, `bash`, `write`, `jac_ast_edit`, pi-mcp-adapter
+ the 174 jac MCP tools. Note `defaultTools` is a global allowlist — extension
tools must be named in it or they silently vanish. No `edit`, no home
packages/skills/themes.

## Setup

Clone side by side (the package reference `../../pi-jac-ast-edit` is relative
to `.pi/settings.json`):

```bash
git clone git@github.com:chess10kp/jac-pi.git ~/repos/jac-pi
git clone git@github.com:chess10kp/pi-jac-ast-edit.git ~/repos/pi-jac-ast-edit
```

Build the engine binding once:

```bash
cd ~/repos/pi-jac-ast-edit
python3 -m venv .venv
.venv/bin/pip install "tree-sitter>=0.25,<0.27" setuptools
.venv/bin/pip install -e ./python
```

Bootstrap the isolated agent dir (gitignored — it holds auth symlinks, never
commit it):

```bash
mkdir -p ~/repos/jac-pi/.pi-home
ln -sf ~/.pi/agent/auth.json ~/repos/jac-pi/.pi-home/auth.json
ln -sf ~/.pi/agent/models-store.json ~/repos/jac-pi/.pi-home/models-store.json
ln -sf ~/.pi/agent/models.json ~/repos/jac-pi/.pi-home/models.json
cp ~/.pi/agent/APPEND_SYSTEM.md ~/repos/jac-pi/.pi-home/APPEND_SYSTEM.md
cat >> ~/repos/jac-pi/.pi-home/APPEND_SYSTEM.md <<'MD'

## GitHub Actions

- ALWAYS use the `gh` CLI for anything GitHub Actions related: `gh run view`, `gh run watch`, `gh run list`, `gh pr checks`, `gh api`. Never open a browser or use other means.
- Push and watch CI to terminal state before reporting success — never claim a run passed without `gh run view` confirming the conclusion.
MD
printf '{\n  "/home/jac/repos": true\n}\n' > ~/repos/jac-pi/.pi-home/trust.json
cat > ~/repos/jac-pi/.pi-home/mcp.json <<'JSON'
{
  "settings": { "directTools": false },
  "mcpServers": {
    "jac": { "command": "jac", "args": ["mcp"], "enabled": true, "directTools": false }
  }
}
JSON
cat > ~/repos/jac-pi/.pi-home/settings.json <<'JSON'
{
  "defaultProvider": "opencode",
  "defaultModel": "muse-spark-1.2-contributor-free",
  "defaultThinkingLevel": "high",
  "defaultTools": ["read", "bash", "write", "jac_ast_edit"],
  "packages": ["npm:pi-mcp-adapter"]
}
JSON
```

Add the launcher to `~/.zshrc`:

```zsh
jacpy() { PI_CODING_AGENT_DIR="$HOME/repos/jac-pi/.pi-home" pi -e "$HOME/repos/pi-jac-ast-edit/extensions/jac-ast-edit.ts" "$@" }
```

It runs in whatever directory you're in — the agent dir supplies config
(`settings.json`, `mcp.json`, `APPEND_SYSTEM.md`), the `-e` flag loads the
`jac_ast_edit` extension explicitly (project `.pi/` discovery would look in
*your* project, not this one), and your project's own `.pi/` config still
merges in normally.

## Workspace layout

| File | Purpose |
|---|---|
| `.pi/settings.json` | project config (plain `pi` from this dir): loads `../../pi-jac-ast-edit` on top of your global config |
| `.pi/mcp.json` | the `jac` MCP server — syntax/validate/lint/format/run + docs |
| `.pi-home/` | isolated agent dir for `jacpy` (gitignored; bootstrap above) |
| `.pi-home/mcp.json` | jac server definition — agent-dir scoped so it loads from any cwd |
| `.gitignore` | keeps `.pi-home/` and engine build artifacts out of git |

Edits to `~/repos/pi-jac-ast-edit` are picked up in place — `/reload`, no
reinstall. Local-path packages expect the sibling-clone layout above.
