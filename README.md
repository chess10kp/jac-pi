# jac-pi

Dedicated pi workspace for Jac editing. Running `pi` from this directory gives
you the Jac AST-editing toolchain on top of your normal global config.

## What's in the config

| File | Purpose |
|---|---|
| `.pi/settings.json` | Loads [`pi-jac-ast-edit`](../pi-jac-ast-edit) — the `jac_ast_edit` tool (tree-sitter-backed AST editing for `.jac`, harvested from Empryo's ast_edit) |
| `.pi/mcp.json` | Enables the `jac` MCP server (`jac mcp`) — `jac_check_syntax`, `jac_validate_jac`, `jac_lint_jac`, `jac_format_jac`, `jac_run_jac`, docs tools |

## Usage

```bash
cd ~/repos/jac-pi
pi
```

Typical loop: `jac_ast_edit` for symbol surgery → `jac_check_syntax` to verify
→ `jac_format_jac` to canonicalize.

## Isolated mode (`jacpy`)

`jacpy` (zsh function) runs pi fully isolated from your home config via
`PI_CODING_AGENT_DIR=~/repos/jac-pi/.pi-home` — only the Jac stack loads:

- `.pi-home/settings.json` — `pi-mcp-adapter` + default model
- `.pi-home/APPEND_SYSTEM.md` — home comm style + GitHub Actions rule
  (always use `gh` CLI: `gh run view/watch`, `gh pr checks`, `gh api`;
  confirm conclusion before claiming CI passed)
- `.pi-home/auth.json` etc. — symlinks into `~/.pi/agent` (gitignored,
  never commit credentials)
- project `.pi/settings.json` — `pi-jac-ast-edit`
- project `.pi/mcp.json` — the `jac` MCP server

Bootstrap after a fresh clone:

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
cat > ~/repos/jac-pi/.pi-home/settings.json <<'JSON'
{
  "defaultProvider": "opencode",
  "defaultModel": "muse-spark-1.2-contributor-free",
  "defaultThinkingLevel": "high",
  "packages": ["npm:pi-mcp-adapter"]
}
JSON
```

Local-path packages are referenced in place — edits to
`~/repos/pi-jac-ast-edit` take effect after a `/reload` (no reinstall).

**Clone layout:** the package reference `../../pi-jac-ast-edit` is relative to
`.pi/settings.json`, i.e. it expects the sibling checkout
`~/repos/pi-jac-ast-edit`. Clone both repos side by side:

```bash
git clone git@github.com:chess10kp/jac-pi.git ~/repos/jac-pi
git clone git@github.com:chess10kp/pi-jac-ast-edit.git ~/repos/pi-jac-ast-edit
```

Project settings merge with your global `~/.pi/agent` config; start pi with
`--no-plugins` variants or trim global packages if you want this dir fully
minimal.
